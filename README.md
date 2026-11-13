# Home Assistant AlphaESS Modbus & Predbat Integration

This repository provides instructions, configurations, and documentation for integrating an **AlphaESS energy storage system (ESS)** via **Modbus TCP** with **Predbat** in Home Assistant and AppDaemon. 

Based on the community work by Ramon van Raaij (`ha-alphaess-modbus`), this setup maps all necessary registers and pairs them with Predbat's predictive optimization using dynamic energy tariffs (such as Nordpool).

---

## 🚀 Features
* **Direct Modbus TCP Control:** Communicates locally with the AlphaESS inverter over port `502`.
* **Predbat Optimization:** Leverages Predbat's advanced predictive algorithms for cost-effective battery scheduling.
* **Dynamic Tariff Support:** Transforms raw hourly market rates (e.g., via Nordpool) into the required `raw_today` and `raw_tomorrow` schema including taxes and VAT.
* **Local & Privacy-Focused:** Runs fully locally via Home Assistant and AppDaemon without cloud dependencies.

---

## 🛠 1. AlphaESS Modbus Integration

To add all inverter registers to Home Assistant, use the integration by Ramon van Raaij:
* **Repository:** [ramonvanraaij/ha-alphaess-modbus](https://github.com/ramonvanraaij/ha-alphaess-modbus)

### Instructions:
1. Download the repository as a `.zip` file and extract it on your local drive.
2. Use the Home Assistant Studio Code Server app (or File Editor) to drag and drop the content of the zip file into `/config/packages/alphaess` (or `/homeassistant/packages/alphaess`). Create the `packages` directory first if it does not exist.
3. Add the package reference to your `configuration.yaml`:

```yaml
homeassistant:
  packages:
    alphaess: !include integration_alpha_ess.yaml
default_config:

# rest of your config
```

4. Configure your network parameters and host variables (e.g., in `secrets.yaml` or directly):

```yaml
alphaess_modbus_host_ip: "<YOUR_INVERTER_IP>"
alphaess_modbus_host_port: 502
alphaess_modbus_slaveId: 85 # 0x55 - default for AlphaESS
```

### Finding your Inverter IP & Port:
* Set a static IP in your router. Look up the MAC address on the physical sticker of your battery.
* Alternatively, log in to the [AlphaESS Portal](https://portal.alphaess.com/), go to **DEVICES**, select your device, and check **device hotspots** in the submenu.
* **Important:** Ensure communication port `502` is open on your system. This can be enabled via the installer app/portal or by opening a support ticket with AlphaESS customer service.

Test the port locally using the CLI:
```bash
nc -zv <IP-ADDRESS-INVERTER> 502
```

After restarting Home Assistant, verify the integration works by adding the dashboard card included in the integration (`alphaess_view.yaml`).

---

## ⚙️ 2. AppDaemon Cleanup & Predbat Configuration

Predbat runs inside AppDaemon (e.g., path: `/app_configs/<ALPHA NUMERIC>_predbat/apps.yaml`). 

### Cleanup Default Files
Before running Predbat, ensure you remove conflicting or default files from your AppDaemon `apps/` directory that cause startup errors:
* **`givenergy.yaml`** (Predbat's default inverter template)
* **`hello.py`** (AppDaemon's default test app)

### Apps.yaml Configuration
Configure your `apps.yaml` to point to your import/export rate sensors and specify your inverter settings:

```yaml
predbat:
  module: predbat
  class: PredBat

  # --- INVERTER CONFIGURATION ---
  inverter_type: "AlphaESS_Modbus"
  inverter:
    has_ge_inverter_mode: false
    has_discharge_enable_time: true

  # --- NORDPOOL IMPORT AND EXPORT RATES ---
  metric_octopus_import: sensor.predbat_import_rates
  metric_octopus_export: sensor.predbat_export_rates
  metric_standing_charge: 0
```

---

## 📊 3. Dynamic Tariffs (`Predbat Import Rates`)

Because raw Nordpool price data format isn't natively supported out-of-the-box by Predbat, a template sensor is required to compute all-in costs (market price + import costs + energy tax, multiplied by VAT) and format them into the required `raw_today` and `raw_tomorrow` schema.

### Setup Steps:
1. Locate your Nordpool integration config entry ID (go to the Nordpool integration, click the 3 dots, and select **"Copy Item ID"**).
2. Identify your area code (e.g., `NL`) and resolution (e.g., `60` minutes).
3. Add the following template sensor to your `configuration.yaml` (adjusting helper input entities and your Nordpool sensor names accordingly):

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

> **Note:** Raw integer prices from Nordpool like `220` represent 22.0 cents, hence the division by `1000` to convert correctly to Euros per kWh.

### Expected Output Format
The resulting sensor outputs attributes conforming to this layout:
```yaml
raw_today:
  - start: '2026-11-12T23:00:00+00:00'
    end: '2026-11-13T00:00:00+00:00'
    value: 0.28408
    value_inc_vat: 0.34374
raw_tomorrow:
  - start: '2026-11-13T23:00:00+00:00'
    end: '2026-11-14T00:00:00+00:00'
    value: 0.328
    value_inc_vat: 0.39688
unit_of_measurement: EUR/kWh
friendly_name: Predbat Import Rates
```

---

## 🔄 4. Dispatch Automation

The final step is to create an automation that bridges the commands generated by Predbat and dispatches them to your AlphaESS Modbus integration registers. Check the repository example scripts to match your specific entity IDs.

---

## 📄 License
This project is open source.
