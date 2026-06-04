SMACC – SMA Charge Control
SMACC is a Home Assistant package for planned, explainable battery charge control on SMA-based systems.
It is meant for setups where Home Assistant knows enough about:
current battery state of charge
current PV production
household consumption
grid import/export
short-term PV forecast
target battery levels for midday and end of day
…and should then decide when charging should start, when it should wait, and when charging must be blocked.
SMACC also adds diagnostics, helper entities, dashboard-friendly status sensors and Modbus write logic for the inverter / battery controller.
---
Why SMACC exists
Most SMA/BYD/Home-Assistant setups can already show a lot of energy data, but they usually do not provide a clear high-level charging strategy out of the box.
Typical problems are:
the battery charges too early although enough PV is expected later
the battery starts too late and misses the target SOC
forecast values alone are too nervous for a stable start-time decision
it is hard to see why charging is currently allowed or blocked
Modbus control exists, but the decision logic around it is missing
SMACC solves that by combining:
live measurements
forecast data
historical time-slot averages
target-based charging rules
transparent status / decision sensors
The goal is not just to charge the battery somehow, but to do it in a way that is:
traceable
adjustable
stable enough for daily use
visible in Home Assistant
---
What SMACC does
In plain language, SMACC does the following:
1. It determines the current energy situation
SMACC derives the current house consumption from:
PV production
grid import
grid export
battery charge power
battery discharge power
That gives it a better internal view of the real house load than relying on a single raw sensor alone.
2. It calculates charging targets
SMACC works with multiple battery targets, for example:
a midday target
an end-of-day target
optional night reserve / morning forecast adjustments
optional balancing target (e.g. 100%)
3. It determines when charging should start
The current version uses a 3-day time-slot approach for the day-end charging start.
Instead of only asking:
> “What does the next 2h forecast say?”
SMACC asks:
> “At the current time of day, what usually happened during the same 30-minute slot across the last 3 days?”
That is used to build:
a PV time-slot average
a house-consumption time-slot average
an expected charging power for that slot
a derived charge duration until the target SOC
a calculated latest sensible charging start time
4. It decides whether charging should be allowed or blocked
From the calculated state, SMACC derives decisions such as:
target reached → block charging
only midday target relevant → allow charging
charging phase active → continue charging
enough time remains → delay charging
start time reached / target time getting tight → begin charging
5. It writes charging permissions via Modbus
SMACC ultimately controls the SMA/CmpBMS side by writing Modbus registers.
That means it is not just a display package. It is an active control package.
6. It exposes readable helper and diagnostic entities
SMACC creates sensors such as:
current decision text
current effective charging target
expected charging power
required charging duration
charge start forecast for current target
charge start forecast for day-end target
target reached protection state
“battery charging despite block” detection
These make it much easier to debug the logic in Home Assistant.
---
Important design principle
This package is not plug-and-play by entity name.
Your entity IDs will almost certainly be different.
That means:
the logic is reusable
the exact entity IDs are not
So the important thing for installation is not:
> “Do I have exactly the same entity names?”
but:
> “Do I have entities that represent the same meaning?”
This README therefore describes the required entity roles and integration types, not just the original local names.
---
Home Assistant requirements
Core features required
SMACC expects a normal YAML-based Home Assistant setup with these core capabilities:
`packages`
`template`
`automation`
`script`
`timer`
`input_boolean`
`input_number`
`input_datetime`
`input_select`
`input_text`
`statistics`
Enable packages for example like this:
```yaml
homeassistant:
  packages: !include_dir_named packages
```
---
Required integrations
1. Recorder
Required
Recorder is needed because SMACC uses Home Assistant statistics history, especially for the 3-day time-slot logic.
The current implementation reads from:
`statistics_short_term`
That means Recorder must be enabled and the relevant source sensors must actually produce statistics.
---
2. SQL
Required
SMACC uses SQL sensors to calculate historical 30-minute time-slot averages from the last 3 days.
These are used for example for:
PV time-slot average
house-consumption time-slot average
Without SQL, the current 3-day charge-start logic will not work as designed.
---
3. Modbus
Required for active control
SMACC writes charging states back to the SMA system via `modbus.write_register`.
Without Modbus, SMACC can still be adapted into a display/decision package, but it will not actively control charging.
---
4. PV forecast integration
Required
SMACC needs hourly forecast data. The original setup uses multiple forecast entities per PV surface / roof area.
You can use any forecast source as long as you can provide equivalent information.
Examples of suitable sources can include:
Forecast.Solar
Solcast
another PV forecast integration
your own forecast template sensors
The important thing is not the product name. The important thing is that Home Assistant exposes the needed forecast roles.
---
5. Optional EV / wallbox logic
Optional
SMACC can take EV charging context into account. That part is optional, but the current package already contains hooks for it.
If you do not need EV-aware behavior, this part can be simplified or removed.
---
Required entity roles
Below are the roles SMACC needs. You must map these roles to your own entity IDs.
A. Battery and live power flow
These roles are mandatory.
Role	Unit	Meaning
Battery SOC	`%`	Current state of charge of the home battery
Battery charge power	`W`	Current charging power of the battery
Battery discharge power	`W`	Current discharge power of the battery
Total PV production	`W`	Current total PV generation
Grid import	`W`	Current import from grid
Grid export	`W`	Current export to grid
SMACC uses these values to derive the current house load and determine whether charging is active, blocked or still needed.
---
B. PV forecast roles
These are mandatory as well.
SMACC currently expects forecast information for up to three PV surfaces / sub-systems.
For each surface, it expects these role types:
Role	Unit	Meaning
Current hour forecast	`kWh`	Forecast energy for the current hour
Next hour forecast	`kWh`	Forecast energy for the next hour
Remaining production today	`kWh`	Forecast remaining PV energy for the rest of today
Total production today	`kWh`	Forecast PV energy for the full current day
Total production tomorrow	`kWh`	Forecast PV energy for the full next day
Important
If your setup has:
only one PV surface
only two PV surfaces
or more than three
then you must adapt the package templates accordingly.
The provided package is built around a three-surface example layout.
So for another installation, the package must be adjusted to your actual forecast structure.
---
C. Optional EV / wallbox roles
These are optional, but present in the current SMACC package.
Role	Unit	Meaning
EV lock active	state	Whether EV-related battery lock logic is enabled
EV lock should be active	state	Derived binary state for EV lock logic
EV charge mode	text/state	Current charger mode
EV charge power	`W`	Current EV charging power
House consumption without EV	`W`	House load with EV share removed
PV deficit caused by EV	`W`	EV-related PV deficit
Target battery discharge power for EV logic	`W`	Used when writing Modbus discharge limits
If you do not need EV-aware behavior, document that clearly and remove or replace these references.
---
Required helper categories created by SMACC
SMACC creates its own Home Assistant helpers and derived entities. These do not need to exist in advance; they are created by the package.
Main categories include:
Control helpers
booleans for enabling/disabling logic
a “charge phase active” boolean
balancing control booleans
dynamic target-time flags
Numeric parameters
midday target SOC
end-of-day target SOC
hysteresis
battery capacity
efficiency
max charge power
max discharge power
reserve times
forecast/consumption factors
Time helpers
earliest allowed charge-phase start
latest allowed charge-phase end
desired target time
remembered PV start/end timestamps
Status / diagnostic sensors
current decision text
current effective charge target
charge-start forecast
required charge duration
target-reached block state
PV standby state
“battery charging despite block” indicator
3-day time-slot sensors
PV time-slot mean over last 3 days
house-consumption time-slot mean over last 3 days
expected charging power derived from those values
---
Modbus requirements
SMACC is written for a setup where Home Assistant can control the battery/inverter through a Modbus hub.
In the original package, the expected Modbus definition is conceptually:
one Modbus hub for the SMA inverter / battery interface
one slave device representing the CmpBMS control endpoint
writable registers for charge/discharge control
Register roles used by the package
Register	Meaning
`40793`	Minimum charge power
`40795`	Maximum charge power
`40797`	Minimum discharge power
`40799`	Maximum discharge power
`40801`	Grid setpoint
`40236`	Operating mode
Operating modes used in the package
Value	Meaning
`1438`	Charging allowed / automatic mode
`2290`	Charging blocked / discharge mode
Very important
These registers are setup-specific knowledge.
If another installation uses:
another inverter model
another slave id
another BMS integration path
another Modbus register layout
then the Modbus block must be adapted carefully before use.
---
How the 3-day charge-start logic works
The package no longer relies only on a short-term “next 2h” value for day-end charge start.
Instead it calculates:
a PV mean for the current 30-minute slot over the last 3 days
a house-consumption mean for the same slot over the last 3 days
an expected net charging power from those two values
required charging duration until target SOC
a calculated latest sensible charge start time
Why this is useful
This makes the charge-start logic less nervous when:
the 2h forecast suddenly drops or spikes
house-consumption averages are temporarily misleading
momentary live values would trigger premature charging
In short:
2h forecast is still useful as a comparison signal
3-day time-slot average becomes the more stable planning signal for the day-end target
---
Installation and setup
1. Back up your existing setup
Before installing SMACC, back up:
your current battery package
Home Assistant configuration
dashboard configuration
Modbus configuration
any automation currently controlling battery charging
---
2. Copy the package file into your packages directory
Example:
```text
/config/packages/smacc.yaml
```
---
3. Open the package and adapt all environment-specific references
This is the most important step.
You must adapt at least:
Entity IDs
Replace the example entity IDs with your own entities for:
battery SOC
battery charge/discharge power
PV production
grid import/export
PV forecast per surface
optional EV-related sensors
PV forecast structure
Adapt the templates if your installation has:
fewer than three PV surfaces
more than three PV surfaces
a different forecast entity layout
Modbus configuration
Adapt:
Modbus hub name
slave id
registers
operating mode values
if your system differs from the original example.
---
4. Verify Recorder and statistics support
Before restart, make sure:
Recorder is active
the relevant measurement sensors are eligible for statistics
Home Assistant actually records those statistics
This matters especially for:
total PV power
internally derived house-consumption values
---
5. Restart Home Assistant
A full restart is the safest method after:
package changes
SQL sensor additions
Recorder/statistics-related logic changes
---
6. Check that SMACC entities are created
After restart, verify that core SMACC entities exist, especially:
decision text
effective target SOC
charge-start sensors
required charge duration
PV time-slot mean (3 days)
house-consumption time-slot mean (3 days)
expected charging power from 3-day logic
charge status helper
main automations
---
7. Allow time for historical learning
The 3-day time-slot logic depends on history.
What to expect
immediately after installation: package can work, but history-based values may still be weak
after some hours: first statistics become more useful
after roughly 3 days: the time-slot means become properly representative
So a fresh install should not be judged only by the first few minutes.
---
Recommended verification checklist
After installation, check the following:
Core system
[ ] Recorder is running
[ ] SQL sensors are available
[ ] Modbus write service works
[ ] Package is loaded without YAML errors
Required live data
[ ] battery SOC updates correctly
[ ] PV production updates correctly
[ ] grid import/export updates correctly
[ ] battery charge/discharge power updates correctly
Forecast data
[ ] current-hour forecast values are present
[ ] next-hour forecast values are present
[ ] remaining-today forecast values are present
[ ] tomorrow forecast values are present
SMACC logic
[ ] decision text changes plausibly
[ ] target SOC is plausible
[ ] charge-start calculation is plausible
[ ] 3-day PV time-slot mean has values
[ ] 3-day house-consumption mean has values
[ ] expected charging power is plausible
Modbus control
[ ] charging allow/block writes behave as expected
[ ] no wrong mode is written during standby
[ ] target-reached blocking works
---
Known limitations and pitfalls
1. Charge-phase flag can survive restarts
The helper
`input_boolean.sma_akku_ladephase_bis_ziel_aktiv`
is persistent.
That means after a reload or restart, a previously active charge phase may still be restored even if the newly calculated charge start would actually be later.
This is important in operation and should be handled explicitly if you want restart-safe behavior.
2. The 3-day logic needs history
The new logic is intentionally history-based.
Without sufficient Recorder/statistics history, the time-slot logic is less meaningful at first.
3. Example entity names are not portable
The package logic can be portable.
The original entity names are not.
Every installation must do its own mapping.
4. Forecast layout is installation-specific
The original package assumes a specific forecast layout with multiple PV surfaces.
That structure may need adaptation on other systems.
---
Minimal installation checklist
Home Assistant features
[ ] YAML packages enabled
[ ] Recorder enabled
[ ] SQL integration available
[ ] Modbus integration available
Required live sensors
[ ] battery SOC sensor
[ ] battery charge power sensor
[ ] battery discharge power sensor
[ ] total PV production sensor
[ ] grid import sensor
[ ] grid export sensor
Required forecast roles
[ ] current hour forecast per PV surface
[ ] next hour forecast per PV surface
[ ] remaining production today per PV surface
[ ] total production today per PV surface
[ ] total production tomorrow per PV surface
Required actuator side
[ ] Modbus hub configured
[ ] correct slave configured
[ ] writable charge/discharge control registers available
Optional
[ ] EV-aware signals available
[ ] dashboard entities available
[ ] enough Recorder history available
---
Suggested next improvements
If you want to make SMACC easier for reuse by other people, the next good steps would be:
make all site-specific entity IDs configurable
separate EV logic into an optional module
move Modbus hub/slave/register values into a clearly documented config section
add restart protection for the persistent charge-phase flag
split “example mappings” and “core logic” more cleanly
---
Summary
SMACC is useful if you want more than simple battery monitoring.
It is designed for people who want Home Assistant to:
understand battery, PV and house state
delay charging when that makes sense
start charging when the target would otherwise be missed
explain its decision in readable sensors
actively control the inverter/BMS side through Modbus
To use it on another system, the most important task is not copying entity names 1:1, but mapping the required entity roles correctly and adapting the Modbus block to the real hardware.
