# Home Assistant — Motion-Activated Lighting Logic Fix

**Host:** Primary AI Server (Home Assistant)
**Severity:** Low — incorrect automation behavior, no service outage
**Outcome:** Full resolution, light now follows correct sunset/sunrise
and adaptive timeout logic

---

## Incident Summary

An office motion-activated light was turning on at all hours of the
day, ignoring a configured sun-based condition that should have
restricted operation to 30 minutes before sunset through sunrise.

Two separate issues were found, each masking the other:

1. An old background automation built from a Home Assistant
   blueprint had no time or sun conditions and was triggering the
   light on any motion, 24/7, overriding the intended logic entirely.

2. The intended custom automation had a YAML syntax error and an
   incorrectly placed trigger delay that prevented its conditions
   from being evaluated correctly.

---

## Root Cause

**Primary — rogue blueprint automation.**
An older automation named "Office motion detection," built from a
standard Home Assistant blueprint, was still active in the
background. It had no time or sun conditions configured, so it
triggered the light on any motion at any time of day. This silently
overrode the intended automation regardless of how correctly the
intended automation was written.

**Secondary — YAML syntax and logic errors.**
The intended automation used `trigger: state` instead of the
correct `platform: state` key. It also had a three-hour delay
attached to the trigger itself, meaning the sun condition was not
evaluated until three hours after motion was first detected. This
produced unpredictable and seemingly random behavior that did not
correlate with when motion actually occurred.

---

## Why This Was Hard to Diagnose

Two automations were fighting each other, and only one of them was
visibly broken. The custom automation appeared to have a logic bug,
but fixing its YAML alone would not have resolved the issue — the
rogue blueprint automation would have continued overriding it
regardless. Both issues had to be identified before either fix
would produce correct behavior.

This is a common pattern in Home Assistant once multiple
automations accumulate over time — older automations built from
blueprints can remain active and silently conflict with newer,
more deliberate configurations.

---

## Resolution

### Step 1 — Disable the rogue blueprint automation

Identified and disabled the background "Office motion detection"
automation to stop the 24/7 triggering.

### Step 2 — Rebuild the turn-on automation

Corrected the YAML syntax error, removed the three-hour trigger
delay, and set explicit sun-based conditions so the automation
only fires between 30 minutes before sunset and sunrise.

### Step 3 — Build an adaptive turn-off automation

Replaced the previous turn-off logic with a choose block providing
two different timeout durations depending on time of night — a
longer timeout during normal hours to avoid the light turning off
while occupied, and a shorter timeout overnight.

---

## Final Working Configuration

### Automation 1 — Office Motion Turn On

Role: gatekeeper. Only allows the light to turn on between 30
minutes before sunset and sunrise.

```yaml
alias: Office - Motion Turn On
description: "Turns on office light 30 minutes before sunset until sunrise."
trigger:
  - platform: state
    entity_id: binary_sensor.office_sensor_motion
    to: "on"
condition:
  - condition: sun
    after: sunset
    after_offset: "-00:30:00"
  - condition: sun
    before: sunrise
action:
  - action: light.turn_on
    target:
      entity_id: light.office
    data:
      color_temp_kelvin: 6500
      brightness_pct: 100
mode: single
```

### Automation 2 — Office Motion Turn Off

Role: adaptive timer. Turns the light off after motion stops,
using a shorter timeout overnight and a longer timeout during
normal hours.

```yaml
alias: Office - Motion Turn Off
description: "Turns off office light after 1 hour of no motion, or 20 minutes after 2 AM."
trigger:
  - platform: state
    entity_id: binary_sensor.office_sensor_motion
    to: "off"
action:
  - choose:
      - conditions:
          - condition: time
            after: "02:00:00"
            before: "06:00:00"
        sequence:
          - delay:
              minutes: 20
          - action: light.turn_off
            target:
              entity_id: light.office
    default:
      - delay:
          hours: 1
      - action: light.turn_off
        target:
          entity_id: light.office
mode: single
```

---

## Key Lessons

**Conflicting automations can silently override each other.**
A correctly written automation can appear broken when an older,
unrelated automation is overriding its behavior entirely. When
automation logic appears inconsistent with its own configuration,
check for other automations targeting the same entity before
assuming the visible automation is the problem.

**Blueprint-based automations can outlive their usefulness.**
Automations created from blueprints early on can remain active
indefinitely, especially if they were never explicitly disabled
when replaced by custom logic. Periodically audit active
automations for the entities they target.

**`platform` vs `trigger` is a common YAML error.**
Home Assistant trigger blocks use the `platform` key, not
`trigger`. This error can cause an automation to fail silently
or behave unpredictably rather than throwing an obvious error.

**Delays on triggers vs delays in actions are very different.**
A delay attached to a trigger postpones when the automation's
conditions are evaluated — not just when the action runs. This
can cause conditions like sun position to be checked against the
wrong point in time entirely.

**Use `choose` blocks for time-dependent behavior.**
Rather than writing separate automations for different times of
day, a single automation with a choose block can apply different
logic depending on conditions like time of day, providing cleaner
and more maintainable automations.

---

## Diagnostic Checklist for Future Automation Conflicts

1. Identify all automations targeting the affected entity
2. Check each automation for conditions — automations with no
   conditions will fire on every trigger
3. Verify trigger blocks use `platform`, not `trigger`
4. Check whether any delays are attached to triggers vs actions —
   these affect when conditions are evaluated very differently
5. Disable suspect automations one at a time to isolate which
   automation is producing the observed behavior
6. Once isolated, rebuild with explicit conditions and verify
   against expected behavior across different times of day
