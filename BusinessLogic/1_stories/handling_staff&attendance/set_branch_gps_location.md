# Configuring Branch Location for Attendance Confirmation

## Context

As a business grows, managers begin to care not only about whether staff check in, but **where** they check in from.

Without location confirmation, a staff member could:
- check in before arriving,
- check in from another branch,
- or accidentally start work while still commuting.

For some businesses this risk is acceptable.  
For others, especially multi-branch stores, it becomes an operational concern.

Because Modula follows a *“pay for what you use”* philosophy, location confirmation is optional and can be enabled only when a business needs it.

When this capability is enabled, the system needs to know **where the workplace actually is**.

---

## The Situation

A manager enables location-based attendance to ensure staff are physically present at the workplace when starting their shift.

However, the system cannot verify location unless the branch’s workplace location is defined.

The manager needs a simple way to set:
- the branch’s workplace location
- how close staff must be to be considered “at work”

This is not about tracking staff throughout the day.  
It is only about confirming presence at the moment of check-in and check-out.

---

## What the Manager Expects

The manager expects:
- to configure the workplace location once per branch  
- to define a reasonable distance for check-in validation  
- future attendance records to use these settings  
- past attendance records to remain unchanged  

They do **not** expect:
- the system to track staff movement during shifts  
- the system to enforce surveillance or monitoring  
- the change to affect past attendance history  

From their perspective, the goal is simple:  
**confirm presence at the workplace when work begins and ends.**

---

## Constraints & Reality

- Not every business wants location confirmation  
- Branch locations can change over time  
- Attendance records are historical records  
- Location rules must apply only going forward  

This feature must remain optional and safe for history.

---

## How Modula Helps

Modula supports this by allowing managers to:
- define a workplace location for each branch  
- configure an allowed distance for attendance confirmation  
- apply these settings to **future check-ins and check-outs**  
- preserve past attendance records as they were recorded  

This helps businesses build trustworthy attendance records without introducing unnecessary monitoring.

---

## What This Story Is Not About

- Updating branch address or contact information  
- Tracking staff during their shift  
- Managing staff schedules or permissions  

Those belong to other stories.

This story is about **defining the workplace location used for attendance confirmation.**