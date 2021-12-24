---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ


toc_footers:
  - Citycrop LL Driver
  - Firmware — v2.2.1_17112021
  - Hardware — r14.b

includes:
  - errors

search: true

code_clipboard: true

meta:
  - name: description
    content: Documentation for the LL Controller UART API
---

# Introduction

Welcome to the STM32F4xx UART low level API! You can use the API to access various device endpoints, which can get sensor measurements, program a timed load, monitor current load state and more.

The API backend is written onboard in C. The calls and the responses are simple csv messages.

The UART settings are `9600/8-N-1`.

The production version sources are always found [here](https://gitlab.com/citycrop/controller-firmware/-/tree/master/firmware/future) and the corresponding release binaries [here](https://gitlab.com/citycrop/controller-firmware/-/releases).

<aside>👋 Throughout the examples, a csv-styled <code>request</code> is followed by a csv-styled <code>response</code> like so:
</aside>

### example request/response
👉 `set,system_reset,1`

👈 `ok,system_reset`

# Sensor Data

```actionscript
get, sensor_values
```
>👍`ok, value1, value2, ... , value9`

The lower unit returns a total of nine values on demand; these values are refreshed every 1 second.

The `SCD4x` module refresh rate is 5 s. 

### Sensor Response 

Field | Return Range (Units) | Description
--------- | ------- | -----------
pH | 0.0 - 14.0 | pH measured via the probe
EC | 0 ...  (uS) | Electrical Conductivity in micro-Siemens
waterLevel | 0, 25, 50, 75, 100 (%) | Calibrated water tank percentage (extracted from ultrasonic distance measurement)
CO2 | value (ppm) | Carbon dioxide concentration 
temperature | value (C) | Temperature from Pt100 inside EC probe
rH | 0-100 (%) | Relative Humidity
boostCurrent | 0 ~ 400 (mA) | A relatively large value indicates the load (fogger) works
doorStatus | T/F | Door open (holds history)
resetStatus | T/F | Reset pressed (holds history)

The fields are: `pH, EC, waterLevel_CM, CO2, temperature, rH, boostCurrent, doorStatus, resetStatus`

### Sensor Stream

```python
set, sensor_stream, 1
```
>⏰ `ok, sensor_stream, 1`

>👍`ok, value1, value2, ... , value9`

A sensor stream can be activated, so the device spits data every `1 second` automatically.

### Sensor Calibration

```actionscript
get, ph_cal_status
```
```actionscript
get, ec_cal_status
```
> Gets calibration status and returns register with calibration points from EEPROM. Status register is bit-encoded successfully set points

>`ok,status reg:%u,t1:%lu,t2:%lu,t3:%lu` for pH

>`ok,status reg:%u,t1:%lu` for EC


```python
set, ph_cal_clear, 1
```
```python
set, ph_cal_low, 1
```
```python
set, ph_cal_mid, 1
```
```python
set, ph_cal_high, 1
```
```python
set, ec_cal_clear, 1
```
```python
set, ec_cal_single, 1
```
> Sets calibration points and toggles specific bit in status register

```python
set, ph_commit, 1
```
```python
set, ec_commit, 1
```
> Commits and activates configuration

A custom calibration scheme in emulated EEPROM has been implemented; the host has to call a specific set of calibration instructions, after inserting the probe to the appropriate calibration solutions. The host can check the calibration status with `ph_cal_status` and `ec_cal_status`. If the host does not handle the calibration routine, the system will work with predefined values and accuracy will be compromised. 

```c
/**
  * @brief  finds slope with linear regression
  * @param  N   number of samples
  * @param  x[N]    X axis values
  * @param  y[N]    Y axis values
  * @retval slope
  */
double find_slope(int N, const double x[N], const double y[N])
/**
  * @brief  finds intercept with linear regression
  * @param  N   number of samples
  * @param  x[N]    X axis values
  * @param  y[N]    Y axis values
  * @retval intercept
  */
double find_intercept(int N, const double x[N], const double y[N])
```
<aside class="warning">
If host does not perform all the calibration points, the system will not write the values to EEPROM and will not activate the configuration. For finally setting the values to non-volatile memory, host must issue <code>set, ph_commit</code> and <code>set, ec_commit</code>.
</aside>


# Loads


## Toggled Loads 

The PCB controls various different loads, some of them being toggled (0/1) like this:

### Example toggle load
👉 `set,status_led,1`

👈 `ok,status_led,1`

```python
set, status_led, 1  
```
> Sets `status_led` ON

```python
set, solenoid, 0
```
> Sets `solenoid` OFF

```python
set, buzzer, 0
```
```python
set, ultrasonic, 1
```
```python
set, peltier, 2
```
> Runs `peltier` in COLD mode

```python
set, pelt_fan1, 1
```
```python
set, fan2, 1
```
Load | Function | Notes
--------- | ------ | ------
status_led | LED on button
solenoid | 12V Solenoid
buzzer | 12V Alarm
ultrasonic | 24V Boost circuit (fogger)
peltier | H-Bridge driver | 1 for HOT, 2 for COLD
pelt_fan1 | peltier exhaust fan
fan2 | jnsp
<aside>
  👋 All the above return an ack. response in the fashion
  <code>ok, load, action</code>
  <br>
  👋 All the requests are also populated in a <code>get, load</code> fashion which will return the current load status like:
</aside>

### example request toggle status
👉 `get, solenoid`

👈 `ok, solenoid, 1` 


## Timed Loads 

For precice timing of load operation, there exist inner worx that control pump operation with `millisecond` precision, and other timed loads with `seconds` presicion.

### Example timed load
👉 `set,pump_grow,23`

👈 `ok,pump_grow,23`

>Setups a timed load

```python
set, aerator, 5
```
```python
set, watering, 12
```
```python
set, pump_micro, 11
```
```python
set, pump_grow, 5
```
```python
set, pump_bloom, 3
```
```python
set, pump_boost, 4
```
```python
set, pump_ph, 4
```


Load | Function | Notes
--------- | ------ | ------
aerator | Air Feed | seconds 
watering | Watering Pump | seconds 
pump_micro | micro pump | millisecond 
pump_grow | micro pump | millisecond 
pump_bloom | micro pump | millisecond 
pump_boost | micro pump | millisecond 
pump_ph | micro pump | millisecond 


### Time-keeper

>Gets and clears a timed load runtime

```actionscript
get, timer_micro
```
```python
set, timer_micro, 0
```
```actionscript
get, timer_grow
```
```python
set, timer_grow, 0
```
```actionscript
get, timer_bloom
```
```python
set, timer_bloom, 0
```
```actionscript
get, timer_boost
```
```python
set, timer_boost, 0
```
```actionscript
get, timer_ph
```
```python
set, timer_ph, 0
```

An internal timer runs for keeping in non-volatile memory the total time ran for each micro pump. Each micro pump total runtime can be accessed and cleared.



## PWM Loads 

# LED Panel

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get()
```

```shell
curl "http://example.com/api/kittens" \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let kittens = api.kittens.get();
```

> The above command returns JSON structured like this:

```json
[
  {
    "id": 1,
    "name": "Fluffums",
    "breed": "calico",
    "fluffiness": 6,
    "cuteness": 7
  },
  {
    "id": 2,
    "name": "Max",
    "breed": "unknown",
    "fluffiness": 5,
    "cuteness": 10
  }
]
```

This endpoint retrieves all kittens.

### HTTP Request

`GET http://example.com/api/kittens`

### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
include_cats | false | If set to true, the result will also include cats.
available | true | If set to false, the result will include kittens that have already been adopted.

<aside class="success">
Remember — a happy kitten is an authenticated kitten!
</aside>

## Get a Specific Kitten

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get(2)
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get(2)
```

```shell
curl "http://example.com/api/kittens/2" \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let max = api.kittens.get(2);
```

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "name": "Max",
  "breed": "unknown",
  "fluffiness": 5,
  "cuteness": 10
}
```

This endpoint retrieves a specific kitten.

<aside class="warning">Inside HTML code blocks like this one, you can't use Markdown, so use <code>&lt;code&gt;</code> blocks to denote code.</aside>

### HTTP Request

`GET http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the kitten to retrieve

## Delete a Specific Kitten

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.delete(2)
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.delete(2)
```

```shell
curl "http://example.com/api/kittens/2" \
  -X DELETE \
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let max = api.kittens.delete(2);
```

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "deleted" : ":("
}
```

This endpoint deletes a specific kitten.

### HTTP Request

`DELETE http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the kitten to delete

