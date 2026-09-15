# Transform Rule: Datetime to Timezone

## Overview

This transform rule is designed to parse an incoming date or date-time string and convert it to a specific timezone. It intelligently handles two primary input formats:

1.  A full ISO 8601 date-time string (e.g., `2023-10-27T14:30:00Z`).
2.  A simple date string in `yyyy-MM-dd` format (e.g., `2023-10-27`).

The target timezone and, for simple dates, the desired hour of the day can be specified in the transform's attributes. The output is always a standardized ISO 8601 date-time string that includes the correct timezone offset.

### Use Cases

- Ensuring date attributes from an HR source (like `hireDate` or `endDate`) are correctly localized for a target application that requires a specific timezone.
- Standardizing various incoming date formats into a single, timezone-aware format for an identity attribute.
- Converting a simple date (e.g., a user's birthday) into a full date-time at the beginning of the day in their local timezone.

## How to Use in a Transform

To use this rule, create a transform with the `rule` type and reference the rule by name. You can provide optional attributes to control the conversion.

```json
{
  "type": "rule",
  "attributes": {
    "name": "Datetime to Timezone",
    "timezone": "America/New_York",
    "localHourToSet": "8"
  }
}
```

## Configuration Attributes

- `name` (Required): Must be `Datetime to Timezone`.
- `timezone` (Optional):
    - **Description**: The target timezone for the conversion. This should be a valid Olson Time Zone ID (e.g., `America/Los_Angeles`, `Europe/London`, `Australia/Sydney`).
    - **Default**: If omitted, empty, or invalid, the rule defaults to `UTC`.
- `localHourToSet` (Optional):
    - **Description**: An integer from `0` to `23` representing the hour of the day. **This attribute is only used when the input is a simple date (`yyyy-MM-dd`)**. It allows you to set a specific time on that date (e.g., the start of the business day).
    - **Default**: If omitted, empty, or invalid, the rule defaults to `0` (the start of the day, i.e., midnight).

## How It Works in Detail

The rule first checks the format of the `input` string to decide how to process it.

### 1. Full Date-Time Input (e.g., `2024-08-20T17:48:02.123Z`)

- The rule identifies this format by checking for the presence of a `T` separator.
- It parses the input string as a UTC-based `Instant`. **It is critical that the input date-time is in UTC or includes a UTC offset (Z or +/-hh:mm).**
- It converts this `Instant` into a `ZonedDateTime` using the specified `timezone` attribute.
- The `localHourToSet` attribute is ignored in this case.
- The final result is formatted as an ISO 8601 string with the target timezone's offset.

### 2. Simple Date Input (e.g., `2024-08-20`)

- If the input does not look like a full date-time, the rule attempts to parse it as a `LocalDate` (`yyyy-MM-dd`).
- It creates a `ZonedDateTime` by taking the parsed date, setting the time to the beginning of the day (`atStartOfDay`), and applying the specified `timezone`.
- It then adjusts the hour of this `ZonedDateTime` based on the `localHourToSet` attribute.
- The final result is formatted as an ISO 8601 string with the target timezone's offset.

## Examples

### Example 1: Convert a Full UTC Datetime to a Local Timezone

Convert a UTC timestamp from an HR system to the employee's local time in New York.

- **Input Value**: `"2024-09-01T04:00:00Z"`
- **Transform Configuration**:
    ```json
    {
      "type": "rule",
      "attributes": {
        "name": "Datetime to Timezone",
        "timezone": "America/New_York"
      }
    }
    ```
- **Output Value**: `"2024-09-01T00:00:00-04:00"` (The date becomes midnight on Sep 1st in EDT)

### Example 2: Convert a Simple Date to a Specific Timezone

Take a `hireDate` and represent it as the start of the day in the London timezone.

- **Input Value**: `"2024-10-28"`
- **Transform Configuration**:
    ```json
    {
      "type": "rule",
      "attributes": {
        "name": "Datetime to Timezone",
        "timezone": "Europe/London"
      }
    }
    ```
- **Output Value**: `"2024-10-28T00:00:00Z"` (In late October, London is on GMT, which is UTC+0)

### Example 3: Convert a Simple Date and Set a Specific Hour

Take an `endDate` and set the deactivation time to 5 PM (17:00) in the US Pacific timezone.

- **Input Value**: `"2025-01-15"`
- **Transform Configuration**:
    ```json
    {
      "type": "rule",
      "attributes": {
        "name": "Datetime to Timezone",
        "timezone": "America/Los_Angeles",
        "localHourToSet": "17"
      }
    }
    ```
- **Output Value**: `"2025-01-15T17:00:00-08:00"` (The time is set to 5 PM PST)

### Example 4: Invalid Timezone Fallback

An incorrect timezone is provided, so the rule gracefully defaults to UTC.

- **Input Value**: `"2024-07-04T12:00:00Z"`
- **Transform Configuration**:
    ```json
    {
      "type": "rule",
      "attributes": {
        "name": "Datetime to Timezone",
        "timezone": "America/NewYork"
      }
    }
    ```
- **Note**: `America/NewYork` is invalid; it should be `America/New_York`.
- **Output Value**: `"2024-07-04T12:00:00Z"` (The output remains in UTC)

## Important Notes

- If the input value is `null` or an empty string, the transform will return `null`.
- If the input string cannot be parsed as either a valid ISO 8601 date-time or a `yyyy-MM-dd` date, the rule will log an error and return `null`.
- The rule is defensive: invalid values for `timezone` or `localHourToSet` will cause it to fall back to safe defaults (`UTC` and `0`, respectively) rather than failing the transform.
