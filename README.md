# Conectric Gateway for Node.js

Gateway code to receive sensor and EKM meter readings from the Conectric mesh network and forward it to a HTTP POST API and optionally to the Go-IoT BACNET service if configured on your gateway.

Message formats to send to the API, and the format of the API URL used are configured in a file called `clientimpl.js`, described in the "Setup" section.

## Installation

Clone this repo then:

```
cd <directory where repo was cloned to>
npm install
```

Note that if you are upgrading from an older version of this software, you should run `rm -rf node_modules` before running `npm install`.  This ensures that you have all of the dependencies at their expected versions.

## Setup

Plug a Conectric USB router into an available USB port.

Edit `config.json` to set the correct API base URL and other parameters.  Edit `meters.json` to include your EKM v3 and/or v4 meters.

Then:

```
$ npm install
$ npm start
```

## config.json

Contains configurable parameters.

```
{
    "http": {
        "apiUrl": "<API URL to post data to> e.g. https://mydomain.com/api/endpoint",
        "enabled": true
    },
    "go-iot": {
        "enabled": false
    },
    "requestTimeout": <Seconds that a request for a chunk of meter data can run for before being considered timed out, min 1>,
    "maxRetries": <Number of times to retry a request for a chunk of meter data that timed out, min 0>
    "readingInterval": <Seconds between successfully reading a meter and starting the next read, min 1>,
    "useMillisecondTimestamps": <true to use millisecond precision timestamps, false for second precision>,
    "useFahrenheitTemps": <true for Fahrenheit temperature data, false for Celsius>,
    "sendStatusMessages": <true to enable sensor status message processing, false to disable>,
    "sendEventCount": <true to send event count information, false to disable>,
    "sendHopData": <true to send hop count data in messages, false to disable>,
    "useHaystack": <true to use haystack format messages, false to disable>
}
```

You may also add your own extra parameters to the config file.  These can then be referenced in your customized `clientimpl.js` file.

When `go-iot` is enabled, you **must** set `useHaystack` to `true`.  Failure to do this will result in the gateway logging an error message and quitting on startup.

## meters.json

Contains an array of EKM v3 or v4 meters to be read, and information about which RS485 hub each meter is connected to.

```
{
    "meters": [
        {
            "serialNumber": "000300004299",
            "rs485HubId": "0000",
            "version": 4,
            "password": "00000000",
            "ctRatio": 100
        },
        ...
    ]
}
```

The values `password` and `ctRatio` are optional, and used by a separate configuration tool that sets the meter's CT ratio.  Values are as follows:

* `password`: An eight digit number, default meter password is 00000000.
* `ctRatio`: The CT ratio that should be set for the meter, values are between 100 and 5000.

To run this without any meters, use:

```
{
    "meters": [
    ]
}
```

### clientimpl.js

This file exports functions that the gateway code requires you to provide implementations for:

* `formatPayload` should perform any required transformations on the message payload that is to be sent to the API.  If `useHaystack` is `true`, then the message format passed into this function will be the haystack format.
* `buildAPIUrl` should return the full URL of the API endpoint to post a message to.
* `buildAPIHeaders` should return an object describing HTTP headers that will be used when posting a message to the URL generated by `buildAPIUrl`.  If none are required, return an empty object `{}`.
* `getAPIVerb` should return the string `POST` or the string `PUT`.  This will be the HTTP verb used when calling the API.
* `onMeterReadFail`, a callback function that is invoked whenever a meter reading attempt fails due to the meter not responding after retries.  This is passed the serial number of the affected meter.

A stub implementation of each, with examples, is provided in the file `clientimpl_template.js`.  You should copy this to `clientimpl.js` and add your own specific code here.

Both `formatPayload` and `buildAPIUrl` have access to the moment library for date formatting. `buildApiURL` additionally has access to any values that you add to `config.json` so you should put API keys etc in here and reference them in your code using the `config` object. Similarly `buildAPIHeaders` also has access to both the `config` and message payload objects, allowing you to store API keys etc in `config.json` and set headers per message type as needed.

## Sensor Message Formats

When config parameter `useHaystack` is `false` the format of the messages you can expect to receive from each type of Conectric sensor is documented [here](https://www.npmjs.com/package/conectric-usb-gateway-beta), as part of the documentation for the mesh network to USB gateway.

When config parameter `useHaystack` is `true`, message formats will look like the following examples.  In all cases:

* `connection` will be `"conectric"`.
* `networkRef` will be `"conectric"`.
* `device1Ref` will be the gateway ID.
* `device2Ref` will be `null`.
* `id` will be the sensor ID.
* `t` will be ISO-8601 formatted time of the message.

### echoStatus

Note: `battery` will always be `null` as devices are not running from batteries.

```
{ 
  "connection": "conectric",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": null,
  "id": "28b0",
  "gatewayId": "00124b00051422cd",
  "type": "echoStatus",
  "payload": { 
    "battery": null, 
    "eventCount": 236 
  },
  "sensorId": "28b0",
  "sequenceNumber": 229,
  "timestamp": 1588648225536,
  "numHops": 0,
  "maxHops": 0,
  "t": "2020-05-05T03:10:25.536Z"
}

```

### moisture

```
{ 
  "connection": "conectric",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": null,
  "id": "3a4a",
  "gatewayId": "00124b00051422cd",
  "type": "moisture",
  "payload": { 
    "battery": 3.1, 
    "moisture": false, 
    "eventCount": 1 
  },
  "sensorId": "3a4a",
  "sequenceNumber": 2,
  "timestamp": 1588451660099,
  "numHops": 1,
  "maxHops": 0,
  "t": "2020-05-02T20:34:20.099Z"
}
```

### moistureStatus

```
{ 
  "connection": "conectric",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": null,
  "id": "3a4a",
  "gatewayId": "00124b00051422cd",
  "type": "moistureStatus",
  "payload": { 
    "battery": 3.1,
    "moisture": false,
    "temp": 80.2,
    "unit": "F",
    "humidity": 48.32,
    "eventCount": 15 
  },
  "sensorId": "3a4a",
  "sequenceNumber": 16,
  "timestamp": 1588453043581,
  "numHops": 1,
  "maxHops": 0,
  "t": "2020-05-02T20:57:23.581Z" 
}
```

### motion

```
{ "connection": "conectric",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": null,
  "id": "1701",
  "gatewayId": "00124b00051422cd",
  "type": "motion",
  "payload": { 
    "battery": 3.3, 
    "eventCount": 121158, 
    "occupancyIndicator": true 
  },
  "sensorId": "1701",
  "sequenceNumber": 62,
  "timestamp": 1588283168766,
  "numHops": 2,
  "maxHops": 0,
  "t": "2020-04-30T21:46:08.766Z"
}
```

### motionStatus

```
{ 
  "connection": "conectric",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": null,
  "id": "15e0",
  "gatewayId": "00124b00051422cd",
  "type": "motionStatus",
  "payload": { 
    "battery": 3.2, 
    "eventCount": 56216, 
    "occupancyIndicator": false 
  },
  "sensorId": "15e0",
  "sequenceNumber": 169,
  "timestamp": 1588647505086,
  "numHops": 1,
  "maxHops": 0,
  "t": "2020-05-05T02:58:25.086Z" 
}
```

Note: `occupancyIndicator` will always be `false`.

### pulse

```
{ 
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "connection": "conectric",
  "device2Ref": null,
  "id": "1413",
  "gatewayId": "00124b00051422cd",
  "type": "pulse",
  "payload": { 
    "battery": 3.1, 
    "pulse": true, 
    "eventCount": 151 
  },
  "sensorId": "1413",
  "sequenceNumber": 86,
  "timestamp": 1588546855788,
  "numHops": 0,
  "maxHops": 0,
  "t": "2020-05-03T23:00:55.788Z" 
}
```

### pulseStatus

```
{ 
  "connection": "conectric",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": null,
  "id": "dbad",
  "gatewayId": "00124b00051422cd",
  "type": "pulseStatus",
  "payload": { 
    "battery": 3, 
    "eventCount": 0, 
    "pulse": false 
  },
  "sensorId": "dbad",
  "sequenceNumber": 122,
  "timestamp": 1588647605806,
  "numHops": 1,
  "maxHops": 0,
  "t": "2020-05-05T03:00:05.806Z" 
}
```

Note: `pulse` will always be `false`.

### rs485Status

```
{ 
  "connection": "conectric",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": null,
  "id": "1d0c",
  "gatewayId": "00124b00051422cd",
  "type": "rs485Status",
  "payload": { 
    "battery": null, 
    "eventCount": 922239 
  },
  "sensorId": "1d0c",
  "sequenceNumber": 39,
  "timestamp": 1588648178159,
  "numHops": 2,
  "maxHops": 0,
  "t": "2020-05-05T03:09:38.159Z" 
}
```

### switch

```
{ 
  "connection": "conectric",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": null,
  "id": "3d94",
  "gatewayId": "00124b00051422cd",
  "type": "switch",
  "payload": { 
    "battery": 3.3, 
    "eventCount": 152, 
    "switch": true 
  },
  "sensorId": "3d94",
  "sequenceNumber": 251,
  "timestamp": 1588553778855,
  "numHops": 3,
  "maxHops": 0,
  "t": "2020-05-04T00:56:18.855Z"
}
```

### switchStatus

```
{ 
  "connection": "conectric",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": null,
  "id": "1413",
  "gatewayId": "00124b00051422cd",
  "type": "switchStatus",
  "payload": { 
    "battery": 3, 
    "eventCount": 36, 
    "switch": false 
  },
  "sensorId": "1413",
  "sequenceNumber": 138,
  "timestamp": 1588218274974,
  "numHops": 0,
  "maxHops": 0,
  "t": "2020-04-30T03:44:34.974Z" 
}
```

Note: `switch` will indicate the current status of the switch.

### tempHumidity

```
{ 
  "connection": "conectric",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": null,
  "id": "3457",
  "gatewayId": "00124b00051422cd",
  "type": "tempHumidity",
  "payload":
  { 
    "eventCount": 126407,
    "battery": 2.8,
    "humidity": 49.48,
    "temp": 76.5,
    "unit": "F" 
  },
  "sensorId": "3457",
  "sequenceNumber": 152,
  "timestamp": 1588218027201,
  "numHops": 0,
  "maxHops": 0,
  "t": "2020-04-30T03:40:27.201Z" 
}
```

### tempHumidityAdc

```
{ 
  "connection": "conectric",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": null,
  "id": "4443",
  "gatewayId": "00124b00051422cd",
  "type": "tempHumidityAdc",
  "payload": { 
    "battery": 3,
    "eventCount": 411318,
    "humidity": 49.24,
    "adcIn": "008a",
    "adcMax": "07ff",
    "temp": 74.12,
    "unit": "F" 
  },
  "sensorId": "4443",
  "sequenceNumber": 171,
  "timestamp": 1588218032173,
  "numHops": 0,
  "maxHops": 0,
  "t": "2020-04-30T03:40:32.173Z"
}
```

### tempHumidityLight

```
{ 
  "connection": "conectric",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": null,
  "id": "15eb",
  "gatewayId": "00124b00051422cd",
  "type": "tempHumidityLight",
  "payload": { 
    "battery": 3,
    "eventCount": 144848,
    "humidity": 51.22,
    "bucketedLux": 0,
    "temp": 76.23,
    "unit": "F",
    "lightLevel": 19 
  },
  "sensorId": "15eb",
  "sequenceNumber": 67,
  "timestamp": 1588218063484,
  "numHops": 0,
  "maxHops": 0,
  "t": "2020-04-30T03:41:03.484Z"
}
```

## Meter Message Formats

Messages received from meters are formatted as follows.  The fields:

* `gatewayId`
* `type`
* `meterModel`
* `timestamp`
* `sensorId`
* `sequenceNumber`

are all as described [here](https://www.npmjs.com/package/conectric-usb-gateway-beta) as part of the documentation for the mesh network to USB gateway.

### Formatting (Haystack Mode off)

The format for meter messages is dependent on the `useHaystack` configuration setting in `config.json`.  When this is set to `false`, the following message format will be generated.

In common with the sensor messages, meter messages contain a `payload` object with a common schema regardless of the type of smart meter hardware used.  An example message from an EKM v4 meter is shown below:

```
{ 
  "gatewayId": "00124b00051422cd",
  "type": "meter",
  "meterModel": "ekm4",
  "timestamp": 1589326058877,
  "sensorId": "1d0c",
  "sequenceNumber": 76,
  "payload": { 
    "battery": 3.2,
    "kwh_scale": 2,
    "model": "4132",
    "firmware": 25,
    "meter_address": "000300002255",
    "kwh_tot": 6334.78,
    "reactive_energy_tot": 1455.51,
    "rev_kwh_tot": 0,
    "kwh_ln_1": 3012.84,
    "kwh_ln_2": 3321.08,
    "kwh_ln_3": 0,
    "rev_kwh_ln_1": 0,
    "rev_kwh_ln_2": 0,
    "rev_kwh_ln_3": 0,
    "resettable_kwh_tot": 6334.78,
    "resettable_rev_kwh_tot": 0,
    "rms_volts_ln_1": 119.2,
    "rms_volts_ln_2": 119.3,
    "rms_volts_ln_3": 0,
    "amps_ln_1": 2.6,
    "amps_ln_2": 4.6,
    "amps_ln_3": 0,
    "rms_watts_ln_1": 198,
    "rms_watts_ln_2": 510,
    "rms_watts_ln_3": 0,
    "rms_watts_tot": 708,
    "power_factor_ln_1": "C0.92",
    "power_factor_ln_2": "C0.96",
    "power_factor_ln_3": "C0.00",
    "reactive_pwr_ln_1": 86,
    "reactive_pwr_ln_2": 146,
    "reactive_pwr_ln_3": 0,
    "reactive_pwr_tot": 232,
    "line_freq": 59.99,
    "pulse_cnt_1": 0,
    "pulse_cnt_2": 1,
    "pulse_cnt_3": 1,
    "state_inputs": 0,
    "state_watts_dir": 1,
    "state_out": 1,
    "meter_time": "20051304072326",
    "kwh_tariff_1": 37946.6,
    "kwh_tariff_2": 25401.3,
    "kwh_tariff_3": 0,
    "kwh_tariff_4": 0,
    "rev_kwh_tariff_1": 0,
    "rev_kwh_tariff_2": 0,
    "rev_kwh_tariff_3": 0,
    "rev_kwh_tariff_4": 0,
    "cos_theta_ln_1": 0.92,
    "cos_theta_ln_2": 0.96,
    "cos_theta_ln_3": 0,
    "rms_watts_max_demand": 4500,
    "max_demand_period": 1,
    "pulse_ratio_1": 1,
    "pulse_ratio_2": 1,
    "pulse_ratio_3": 1,
    "ct_ratio": 200,
    "auto_reset_max_demand": 0,
    "pulse_output_ratio": 800 
  } 
}
```

`timestamp` is the time that the gateway received the message.  `meter_time` is the time according to the meter when the message was generated.  `meter_time` is as received from the hardware and is formatted as follows:

`YYMMDDXXHHMMSS`

* `YY` = 2 digit year.
* `MM` = month.
* `DD` = day.
* `XX` = day of week, 01 = Sunday, 02 = Tuesday ... 06 = Saturday.
* `HH` = hour of day (24hr clock).
* `MM` = minutes.
* `SS` = seconds.

EKM Omnimeter v3 and v4 meters are supported.  The `meterModel` field will be set to `ekm3` for the v3 Omnimeter, and `ekm4` for the v4.

The v3 Omnimeter does not have all of the features of the v4 and uses a slightly different schema.  Here's an example message from a v3 meter:

```
{ 
  "gatewayId": "00124b00051422cd",
  "type": "meter",
  "meterModel": "ekm3",
  "timestamp": 1589235150812,
  "sensorId": "22fb",
  "sequenceNumber": 55,
  "payload": { 
    "battery": 3.2,
    "model": "4130",
    "firmware": 23,
    "meter_address": "000010007076",
    "kwh_tot": 9583.7,
    "kwh_tariff_1": 5931.3,
    "kwh_tariff_2": 3652.4,
    "kwh_tariff_3": 0,
    "kwh_tariff_4": 0,
    "rev_kwh_tot": 9499.3,
    "rev_kwh_tariff_1": 5879,
    "rev_kwh_tariff_2": 3620.3,
    "rev_kwh_tariff_3": 0,
    "rev_kwh_tariff_4": 0,
    "rms_volts_ln_1": 118,
    "rms_volts_ln_2": 0,
    "rms_volts_ln_3": 0,
    "amps_ln_1": 0.4,
    "amps_ln_2": 0,
    "amps_ln_3": 0,
    "rms_watts_ln_1": 42,
    "rms_watts_ln_2": 0,
    "rms_watts_ln_3": 0,
    "rms_watts_tot": 42,
    "cos_theta_ln_1": 0.78,
    "cos_theta_ln_2": 0,
    "cos_theta_ln_3": 0,
    "max_demand": 6635,
    "max_demand_period": 1,
    "meter_time": "20051203061051",
    "ct_ratio": 200,
    "pulse_cnt_1": 0,
    "pulse_cnt_2": 0,
    "pulse_cnt_3": 0,
    "pulse_ratio_1": 0,
    "pulse_ratio_2": 0,
    "pulse_ratio_3": 0,
    "state_inputs": 0 
  } 
}
```

### Formatting (Haystack Mode on)

When haystack mode is on (`useHaystack` in `config.json` set to `true`), messages will look like the following.  Note that `battery` will always be `null` (devices are powered from the meter, not a battery).

EKM v4 Meter:

```
{ 
  "gatewayId": "00124b00051422cd",
  "type": "meter",
  "meterModel": "ekm4",
  "timestamp": 1588391740085,
  "sensorId": "1d0c",
  "sequenceNumber": 249,
  "connection": "ekm",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": "1d0c",
  "equip": "elec_meter",
  "t": "2020-05-02T11:51:29.000Z",
  "id": "000300002255",
  "payload": { 
    "battery": null,
    "volt_A": 119.7,
    "volt_B": 119.6,
    "volt_C": 0,
    "current_A": 0.2,
    "current_B": 3.8,
    "current_C": 0,
    "power_A": 0.006,
    "power_B": 0.418,
    "power_C": 0,
    "reactive_power_A": 0.032,
    "reactive_power_B": 0.12,
    "reactive_power_C": 0,
    "pf_A": 0.19,
    "pf_B": 0.96,
    "pf_C": 0,
    "power": 0.424,
    "power_reactive": 0.152,
    "state_current_dir": 1,
    "freq": 60.03,
    "power_max": 4.5,
    "power_max_period": 1,
    "power_max_auto_reset": 0,
    "energy_A": 3005.56,
    "energy_B": 3204.98,
    "energy_C": 0,
    "energy": 6211.39,
    "power_reactive_h": 1414.06,
    "energy_tariff_1": 37217.8,
    "energy_tariff_2": 24896.1,
    "energy_tariff_3": 0,
    "energy_tariff_4": 0,
    "energy_export_A": 0,
    "energy_export_B": 0,
    "energy_export_C": 0,
    "energy_export": 0,
    "energy_export_tariff_1": 0,
    "energy_export_tariff_2": 0,
    "energy_export_tariff_3": 0,
    "energy_export_tariff_4": 0,
    "energy_resettable": 6211.39,
    "energy_export_resettable": 0,
    "state_pulse_inputs": 0,
    "state_out_cmd": 1,
    "pulse_hisTotalized_1": 0,
    "pulse_hisTotalized_2": 1,
    "pulse_hisTotalized_3": 1,
    "pulse_ratio_1": 1,
    "pulse_ratio_2": 1,
    "pulse_ratio_3": 1,
    "pulse_output_ratio": 800,
    "sensor_ct_ratio": 200,
    "decimal": 2,
    "meter_time": "2020-05-02T11:51:29.000Z" 
  } 
}
```

EKM v3 Meter:

```
{ 
  "gatewayId": "00124b00051422cd",
  "type": "meter",
  "meterModel": "ekm3",
  "timestamp": 1588218048516,
  "sensorId": "22fb",
  "sequenceNumber": 8,
  "connection": "ekm",
  "networkRef": "conectric",
  "device1Ref": "00124b00051422cd",
  "device2Ref": "22fb",
  "equip": "elec_meter",
  "t": "2020-04-30T11:39:11.000Z",
  "id": "000010007076",
  "payload": { 
    "battery": null,
    "volt_A": 118.7,
    "volt_B": 0,
    "volt_C": 0,
    "current_A": 0.4,
    "current_B": 0,
    "current_C": 0,
    "power_A": 0.042,
    "power_B": 0,
    "power_C": 0,
    "reactive_power_A": null,
    "reactive_power_B": null,
    "reactive_power_C": null,
    "pf_A": 0.78,
    "pf_B": 0,
    "pf_C": 0,
    "power": 0.042,
    "power_reactive": null,
    "state_current_dir": null,
    "freq": null,
    "power_max": 6.635,
    "power_max_period": 1,
    "power_mac_auto_reset": null,
    "energy_A": null,
    "energy_B": null,
    "energy_C": null,
    "energy": 9571.7,
    "power_reactive_h": null,
    "energy_tariff_1": 5923.8,
    "energy_tariff_2": 3647.9,
    "energy_tariff_3": 0,
    "energy_tariff_4": 0,
    "energy_export_A": null,
    "energy_export_B": null,
    "energy_export_C": null,
    "energy_export": 9499.3,
    "energy_export_tariff_1": 5879,
    "energy_export_tariff_2": 3620.3,
    "energy_export_tariff_3": 0,
    "energy_export_tariff_4": 0,
    "energy_resettable": null,
    "energy_export_resettable": null,
    "state_pulse_inputs": 0,
    "state_out_cmd": null,
    "pulse_hisTotalized_1": 0,
    "pulse_hisTotalized_2": 0,
    "pulse_hisTotalized_3": 0,
    "pulse_ratio_1": 0,
    "pulse_ratio_2": 0,
    "pulse_ratio_3": 0,
    "pulse_output_ratio": null,
    "sensor_ct_ratio": 200,
    "decimal": null,
    "meter_time": "2020-04-30T11:39:11.000Z" 
  } 
}
```

The v3 Omnimeter does not support all of the fields in the meter payload schema, so you can expect the values of these fields to be null when using a v3 meter:

* `reactive_power_A`
* `reactive_power_B`
* `reactive_power_C`
* `power_reactive_h`
* `freq`
* `power_max_auto_reset`
* `energy_A`
* `energy_B`
* `energy_C`
* `energy_resettable`
* `energy_export_resettable`
* `state_out_cmd`
* `pulse_output_ratio`
* `decimal`

In Haystack mode, the units for each meter data field are as follows:

| Key	                        | Unit          |
| --------------------------- | ------------- |
| `battery`	                  | V             | 
| `temp`	                    | degrees (C/F) |
| `humidity`	                | %             |
| `adcIn`	                    | V             |
| `adcMax`	                  | V             |
| `lightLevel`	              | Lux           |
| `eventCount`	              | #             |
| `volt`	                    | V             |
| `current` 	                | A             |
| `power` (active)	          | kW            |
| `reactive_power`	          | kVAR          |
| `pf` (power factor)	        | %             |
| `frequency`                 |	Hz            |
| `power_max`	                | kW            |
| `energy`	                  | kWh           |
| `power_reactive_h`	        | kVARh         |
| `energy_tarriff`	          | kWh           |
| `energy_export`	            | kWh           |
| `energy_export_tarriff`     |	kWh           |
| `energy_resettable`	        | kWh           |
| `energy_export_resettable`	| kWh           |
| `pulse_hisTotalized`	      | #             |
| `sensor_ct_ratio`           |	A             |