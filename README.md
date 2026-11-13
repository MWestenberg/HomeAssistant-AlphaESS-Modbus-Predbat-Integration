# AlphaESS Modbus integration

Use the following integration from Ramon van Raaij: https://github.com/ramonvanraaij/ha-alphaess-modbus to add all the registers to your HomeAssistant. Ramon did all the grunt work for us figuring out all the registers so give him some love :)

Instructions: just download it as a zip file, extract it on your local drive and use the HA Studio Code Server app to dragg and drop the content of the zipfile into your /config/packages/alphaess directory or if you use the file editor it is the /homeassistant/packages/alphaess directory. If you don't have the packages directory yet you should create it first.

For the integration to work you need to add the following to your homeassistant configuration.yaml:
```yaml
homeassistant:
packages:
  alphaess: !include integration_alpha_ess.yaml
default_config:

# rest of your config
```

Also add the IP address of your AlphaESS

```yaml
# Use this file to store secrets like usernames and passwords.
# Learn more at https://www.home-assistant.io/docs/configuration/secrets/
alphaess_modbus_host_ip: "<YOUR_INVERTER_IP>"
alphaess_modbus_host_port: 502
alphaess_modbus_slaveId: 85 # 0x55 - default for AlphaESS
````

To find out what your ip address is you can either make it static in your router. To find the right MAC address check on your battery, there should be a sticker where it mentions your IP address. Alternative go to https://portal.alphaess.com/ click in the left menu on DEVICES, click on your device (battery) and choose device hotspots in the sub menu. There it will display your IP address. You need to make sure the communication port 502 is open on your system. This can only be done in the installer app or portal. In my case I just opened a ticket with AlphaESS customer service. They can do this as well. 

CLI command to check the port

```bash
nc -zv <IP-ADDRESS-INVERTER> 502
```

After a restart the Modbus integration should work. You can test this by making a card on your dashboard included in the integration (```alphaess_view.yaml```).

# Predbat integration with Nordpool integration

See my apps.yaml and make changes accordingly to your predbat installation. To locate your apps.yaml it depends on your HA setup. In my case I have HASOS and Predbat installed as an App. The location of my apps.yaml is ```/app_configs/<ALPHA NUMERIC>_predbat/apps.yaml```.

There you need to add your sensors to the apps.yaml. Note that Predbat might give errors because it contains by default yaml files that are causing errors like ```givenergy.yaml``` and ```hello.py``` if they are in there.

In order for predbat to work it needs import and export rates. This was such a pain to integrate that I will explain here how I did it. I use Nordpool for my price rates and the format deliverd by Nordpool is not supported out of the box by Predbat.

The configuration I used was the one from Octopus:

```yaml
  #nordpool import en export tarieven
  metric_octopus_import: sensor.predbat_import_rates
  metric_octopus_export: sensor.predbat_export_rates
  metric_standing_charge: 0
```

The sensor.predbat_import_rates and sensor.predbat_export_rates need to be created. For this I created a template action and added it to my configuration.yaml of my HomeAssistant. 

Check the file configuration.yaml to see how the template is build up. 
1. You need to replace your Nordpool config entry in the file with your own: ```<YOUR NORDPOOL CONFIG_ENTRY>```. To find it, go to your integration of Nordpool and click on the 3 dots and select "Copy Item ID".
2. Find your action name. Mine was called nordpool.get_price_indices_for_date. You can test this via developer tools > Actions. Select Get Price Indices for Date and choose your Nordpool Integration. For the today's Date value enter this ```{{ (now() + timedelta(days=1)).strftime("%Y-%m-%d") }}``` set to your area, currency and your resolution ion minutes, mine was 60 min. It should return data that looks like this:
```yaml
NL:
  - start: '2026-11-13T23:00:00+00:00'
    end: '2026-11-14T00:00:00+00:00'
    price: 220
  - start: '2026-11-14T00:00:00+00:00'
    end: '2026-11-14T01:00:00+00:00'
    price: 214.17

 ...

```
Take note of the price here. 220 is actually 22 cents. Later on we need to divide it by 1000.

For tomorrow's date value you can use this: ```{{ (now() + timedelta(days=1)).strftime('%Y-%m-%d') }}```

3. Remember the area code (mine is NL) and replace it in your template marked as \<YOUR AREA\> in the example file. Also note that the resolution is also in there. Mine is set to 60 minutes.
4. Now the tricky part the Import and export rates. This is required to make actual use of Predbat. You need to figure out what the formula is for you. In my case we have the bare market price standard delivered by Nordpool. I needed to add the import cost from my provider, the energy tax that we have here in my country and the VAT. The way it was calculated is:

```
 (market_price + import_cost + eTax) * vat  
```

If your import cost formula is different from your export cost formula you need to have 1 sensor for each as in my example. The market_price comes from your Nordpool integration. Check the name and replace ```<NORDPOOL CURRENT MARKET PRICE SENSOR>``` with your sensor in configuration.yaml

Lets look at the sensor itself. You need to make a sensor for today and one for tomorrow in order for it to work with octopus import in Predbat:
```yaml
sensor:
    - name: "Predbat Import Rates"
      unique_id: predbat_import_rates_nordpool
      unit_of_measurement: "EUR/kWh"
      state: >
        {% set market_price = states('<NORDPOOL CURRENT MARKET PRICE SENSOR>') | float(0) %}
        {% set eTax = states('input_number.energy-tax') | float(0) %}
        {% set vat = states('input_number.vat') | float(0) %}
        {% set import_cost = states('input_number.import-cost') | float(0) %}
        {{ ((market_price + eTax + import_cost) * vat) | round(5) }}
      attributes:
        raw_today: >
          {% set ns = namespace(items=[]) %}
          {% set eTax = states('input_number.energy-tax') | float(0) %}
          {% set vat = states('input_number.vat') | float(0) %}
          {% set import_cost = states('input_number.import-cost') | float(0) %}
          {% if nordpool_today is defined and nordpool_today.<YOUR AREA> is defined %}
            {% for item in nordpool_today.<YOUR AREA> %}
              {% set price = ((item.price / 1000) + eTax + import_cost) | round(5) %}
              {% set price_vat = (price * vat) | round(5) %}
              {% set ns.items = ns.items + [{
                'start': item.start,
                'end': item.end,
                'value': price,
                'value_inc_vat': price_vat
              }] %}
            {% endfor %}
          {% endif %}
          {{ ns.items }}
        raw_tomorrow: >
          {% set ns = namespace(items=[]) %}
          {% set eTax = states('input_number.energy-tax') | float(0) %}
          {% set vat = states('input_number.vat') | float(0) %}
          {% set import_cost = states('input_number.import-cost') | float(0) %}
          {% if nordpool_tomorrow is defined and nordpool_tomorrow.<YOUR AREA> is defined %}
            {% for item in nordpool_tomorrow.<YOUR AREA> %}
              {% set price = ((item.price / 1000) + eTax + import_cost) | round(5) %}
              {% set price_vat = (price * vat) | round(5) %}
              {% set ns.items = ns.items + [{
                'start': item.start,
                'end': item.end,
                'value': price,
                'value_inc_vat': price_vat
              }] %}
            {% endfor %}
          {% endif %}
          {{ ns.items }}
```

I named the sensor: ```Predbat Import Rates``` and the second sensor ```Predbat Import Rates```. in raw_today I set the variables for the calculations to build up my formula. These lines calculate my rates according to the above formula. Note there is division by 1000 of the raw item price because we need to return the value to Predbat in Euro's (0.23) but we are getting an integer value from Nordpool. So 23,0 cents is 230.

When you have configured your sensor properly you need to restart Home Assistant


The result yaml should lhave the following format
```yaml
raw_today:
  - start: '2026-11-12T23:00:00+00:00'
    end: '2026-11-13T00:00:00+00:00'
    value: 0.28408
    value_inc_vat: 0.34374
  ...<repeat based on your interval>
raw_tomorrow:
  - start: '2026-11-13T23:00:00+00:00'
    end: '2026-11-14T00:00:00+00:00'
    value: 0.328
    value_inc_vat: 0.39688
  ...<repeat based on your interval>
unit_of_measurement: EUR/kWh
friendly_name: Predbat Import Rates
```

And that is it. You can test your sensor of course my going to development tools and look in Status for your Predbat Import Rates to see if the resulting yaml is correct. You can now do the same for export rates if you have not already done so.

In predbat as stated above you integrate the prices like this:
```yaml
#NORDPOOL IMPORT AND EXPORT RATES
metric_octopus_import: sensor.predbat_import_rates
metric_octopus_export: sensor.predbat_export_rates
```

5. The final step is to create an automation that dispatches the commands from your Predbat to your AlphaESS Modbus integration. Check the file I added and change the entities where required to the owns you have.

I logged this here because of the amount of trouble I had to go through to figure this out. I hope this has helped you. The result yaml from sensor is actually the generic input Predbat expects and can be applied to other Energy price integrations as well.
