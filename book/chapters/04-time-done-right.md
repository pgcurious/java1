# Chapter 4: Time Done Right

*java.time API, why Date/Calendar was broken, immutability in temporal design*

---

## The Horror Stories

Let me tell you about a bug that cost a company $2.3 million.

A scheduling system stored appointment times as `Date` objects. The UI displayed times correctly. The database stored them correctly. But when appointments crossed midnight in the user's timezone, some were silently moved to the next day. Why? Because a developer wrote `date.setHours(0)` to "clear the time component," not realizing that `Date` stores an instant (milliseconds since Unix epoch), and `setHours` interprets that instant in the JVM's default timezone, which was different from the user's timezone.

Three months of incorrect scheduling. Missed medical appointments. Angry customers.

This is what happens when your date/time API actively works against you.

---

## Why java.util.Date Was Fundamentally Broken

`Date` was one of Java's original classes, released in 1996. Let's examine why it became a cautionary tale:

### Problem 1: Date Isn't a Date

```java
Date today = new Date();
System.out.println(today); // "Wed Dec 25 14:30:00 EST 2024"
```

Despite its name, `Date` represents an **instant**—a specific point on the timeline, stored as milliseconds since January 1, 1970 UTC. It includes both date and time, in no particular timezone (though `toString()` uses the system default, adding to confusion).

When someone asks "what date is it?", the answer depends on timezone. December 25th in Tokyo is still December 24th in New York. But `Date` doesn't carry timezone information—it just stores a moment in time and pretends it's a date.

### Problem 2: Mutability

```java
Date appointment = new Date();
// Pass to some method
calendarService.schedule(appointment);

// Later in that method (or any code with the reference)...
appointment.setTime(0);  // Mutates the original!

// Your appointment is now January 1, 1970
```

Mutable date objects are dangerous. They can be changed by any code that has a reference. This leads to:
- Defensive copying everywhere: `return new Date(internalDate.getTime())`
- Thread-safety nightmares
- Mysterious bugs when dates change unexpectedly

### Problem 3: Insane API Design

```java
Date date = new Date(2024, 12, 25);  // What year is this?
```

This creates a date in the year **3924**. Because the year parameter is offset from 1900. And December is month 11, not 12 (months are 0-indexed). So this is actually month "12" which wraps to January of the next year.

Want January 2024?
```java
Date date = new Date(124, 0, 1);  // Year 124 (from 1900), month 0
```

The Date constructor is so confusing that it was deprecated almost immediately—but millions of codebases still use it.

### Problem 4: Date Can't Do Date Arithmetic

```java
// Add 30 days to a date using java.util.Date
long thirtyDaysInMs = 30L * 24 * 60 * 60 * 1000;
Date later = new Date(date.getTime() + thirtyDaysInMs);
```

This doesn't account for daylight saving time transitions (where days can be 23 or 25 hours), leap seconds, or any other calendar complexity. It's just millisecond arithmetic.

---

## Calendar: The Fix That Made Things Worse

`Calendar` was introduced in Java 1.1 to address Date's shortcomings. It introduced timezones and proper field manipulation:

```java
Calendar cal = Calendar.getInstance();
cal.set(Calendar.YEAR, 2024);
cal.set(Calendar.MONTH, Calendar.DECEMBER);  // At least constants exist
cal.set(Calendar.DAY_OF_MONTH, 25);
```

But `Calendar` had its own problems:

### Still Mutable

```java
Calendar cal = Calendar.getInstance();
someMethod(cal);  // Might mutate it!
```

### Confusing Leniency

```java
Calendar cal = Calendar.getInstance();
cal.set(Calendar.MONTH, 12);  // There is no month 12!
// Does it throw? No. It silently rolls over to January next year.
```

By default, `Calendar` is "lenient"—invalid values roll over instead of failing. This hides bugs.

### Month Still 0-Indexed

Despite being a rewrite, January is still month 0.

### Getting Values Requires Magic Constants

```java
int year = cal.get(Calendar.YEAR);
int month = cal.get(Calendar.MONTH);  // Remember: 0-indexed!
int day = cal.get(Calendar.DAY_OF_MONTH);
int hour = cal.get(Calendar.HOUR_OF_DAY);  // Not HOUR (12-hour clock)
```

---

## Enter java.time: JSR-310

After years of frustration, Java 8 introduced `java.time`, based on the Joda-Time library. The designers started from first principles:

1. **Immutability**: All core classes are immutable and thread-safe
2. **Domain separation**: Different types for different concepts (dates, times, instants, periods)
3. **Clear naming**: `LocalDate`, `LocalTime`, `LocalDateTime`, `ZonedDateTime`
4. **No magic offsets**: January is month 1, year 2024 is 2024
5. **Fluent API**: Method chaining for transformations
6. **Null safety**: Clear semantics for null (usually not allowed)

---

## The Type System: Choosing the Right Type

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         java.time Type Hierarchy                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  "What date is it?"              "What time is it?"                     │
│         ↓                                ↓                              │
│    LocalDate                        LocalTime                           │
│  (2024-12-25)                       (14:30:00)                          │
│         ↓                                ↓                              │
│         └──────────┬─────────────────────┘                              │
│                    ↓                                                    │
│             LocalDateTime                                               │
│         (2024-12-25T14:30:00)                                           │
│        No timezone! Wall clock time.                                    │
│                    ↓                                                    │
│         + ZoneId = ZonedDateTime                                        │
│    (2024-12-25T14:30:00-05:00[America/New_York])                        │
│                    ↓                                                    │
│             Instant (point on timeline)                                 │
│         (2024-12-25T19:30:00Z = milliseconds since epoch)               │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│  "How long between events?"       "How much calendar time?"             │
│         ↓                                ↓                              │
│     Duration                         Period                             │
│  (2 hours, 30 minutes)           (2 years, 3 months, 5 days)            │
│  Machine time, exact              Human time, calendar-based            │
└─────────────────────────────────────────────────────────────────────────┘
```

### LocalDate: Calendar Date Without Time

Use for: birthdays, holidays, due dates—anything that's a calendar date regardless of timezone.

```java
LocalDate christmas = LocalDate.of(2024, 12, 25);
LocalDate christmas2 = LocalDate.of(2024, Month.DECEMBER, 25); // Enum!

LocalDate today = LocalDate.now();
LocalDate tomorrow = today.plusDays(1);
LocalDate nextMonth = today.plusMonths(1);  // Handles month lengths
LocalDate nextYear = today.plusYears(1);    // Handles leap years

int day = christmas.getDayOfMonth();    // 25
Month month = christmas.getMonth();     // DECEMBER
int monthValue = christmas.getMonthValue(); // 12
int year = christmas.getYear();         // 2024
DayOfWeek dow = christmas.getDayOfWeek(); // WEDNESDAY
```

### LocalTime: Time Without Date or Timezone

Use for: store hours, alarm times, "class starts at 9 AM."

```java
LocalTime noon = LocalTime.of(12, 0);
LocalTime precise = LocalTime.of(14, 30, 45, 123_456_789); // h, m, s, nano

LocalTime now = LocalTime.now();
LocalTime later = now.plusHours(2).plusMinutes(30);

int hour = noon.getHour();      // 12
int minute = noon.getMinute();  // 0
```

### LocalDateTime: Date + Time, No Timezone

Use for: "The meeting is at 2024-12-25 14:30"—but be careful! This is **wall clock time**. It doesn't represent a unique instant.

```java
LocalDateTime christmas2pm = LocalDateTime.of(2024, 12, 25, 14, 0);
LocalDateTime fromParts = LocalDateTime.of(
    LocalDate.of(2024, 12, 25),
    LocalTime.of(14, 0)
);

LocalDateTime now = LocalDateTime.now();
```

> **Warning**: `LocalDateTime` is often misused. If you're storing when an event actually happened (a log entry, a transaction), you probably want `Instant` or `ZonedDateTime`. `LocalDateTime` is appropriate for scheduling future events in a user's local time.

### ZonedDateTime: The Full Picture

Use for: exact scheduling, "flight departs Tokyo at 10 AM Tokyo time, arrives LA at 6 AM LA time."

```java
ZoneId tokyo = ZoneId.of("Asia/Tokyo");
ZoneId la = ZoneId.of("America/Los_Angeles");

ZonedDateTime departure = ZonedDateTime.of(
    LocalDateTime.of(2024, 12, 25, 10, 0),
    tokyo
);

ZonedDateTime arrival = ZonedDateTime.of(
    LocalDateTime.of(2024, 12, 25, 6, 0),  // Same calendar day!
    la
);

// How long is the flight?
Duration flightTime = Duration.between(departure, arrival);
// 10 hours (accounts for 17-hour timezone difference)
```

### Instant: Machine Time

Use for: timestamps, when something actually happened, database storage, log entries.

```java
Instant now = Instant.now();
Instant later = now.plusSeconds(60);

// Convert to/from epoch
long epochMilli = now.toEpochMilli();
Instant fromEpoch = Instant.ofEpochMilli(epochMilli);

// Convert to ZonedDateTime for display
ZonedDateTime zdt = now.atZone(ZoneId.of("America/New_York"));
```

`Instant` is the closest thing to `Date`—it's a point on the timeline. But it's immutable, clearly named, and doesn't pretend to be a date.

---

## Duration vs Period: Machine Time vs Human Time

### Duration: Exact Time Span

```java
Duration twoHours = Duration.ofHours(2);
Duration precise = Duration.ofSeconds(3, 500_000_000); // 3.5 seconds

// Arithmetic
Duration total = twoHours.plus(Duration.ofMinutes(30)); // 2h 30m

// Between instants or times
Duration flight = Duration.between(departure, arrival);
long hours = flight.toHours();
long minutes = flight.toMinutes();
```

### Period: Calendar Time Span

```java
Period oneMonth = Period.ofMonths(1);
Period complex = Period.of(1, 2, 3); // 1 year, 2 months, 3 days

// Adding a period to different dates gives different results
LocalDate jan31 = LocalDate.of(2024, 1, 31);
LocalDate feb29 = jan31.plus(oneMonth);  // 2024-02-29 (leap year!)

LocalDate mar31 = LocalDate.of(2024, 3, 31);
LocalDate apr30 = mar31.plus(oneMonth);  // 2024-04-30 (April has 30 days)
```

**Key insight**: A `Duration` of "30 days" is always 30 × 24 × 60 × 60 seconds. A `Period` of "1 month" is context-dependent—it could be 28, 29, 30, or 31 days.

---

## Parsing and Formatting

### ISO-8601: The Default

```java
LocalDate date = LocalDate.parse("2024-12-25");
LocalTime time = LocalTime.parse("14:30:00");
LocalDateTime dt = LocalDateTime.parse("2024-12-25T14:30:00");
ZonedDateTime zdt = ZonedDateTime.parse("2024-12-25T14:30:00-05:00[America/New_York]");
Instant instant = Instant.parse("2024-12-25T19:30:00Z");
```

### Custom Formats

```java
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("MM/dd/yyyy HH:mm");

LocalDateTime dt = LocalDateTime.parse("12/25/2024 14:30", formatter);
String formatted = dt.format(formatter);  // "12/25/2024 14:30"

// Localized formatting
DateTimeFormatter german = DateTimeFormatter
    .ofLocalizedDateTime(FormatStyle.MEDIUM)
    .withLocale(Locale.GERMANY);
String germanDate = dt.format(german);  // "25.12.2024, 14:30:00"
```

---

## The Daylight Saving Time Trap

Twice a year, in most timezones, clocks jump forward or backward. This creates edge cases:

### The Gap: Spring Forward

```java
// In America/New_York, 2024-03-10 02:30 doesn't exist
// Clocks jump from 2:00 AM to 3:00 AM

LocalDateTime doesntExist = LocalDateTime.of(2024, 3, 10, 2, 30);
ZoneId eastern = ZoneId.of("America/New_York");

// What happens?
ZonedDateTime zdt = doesntExist.atZone(eastern);
// 2024-03-10T03:30:00-04:00[America/New_York]
// java.time adjusts to the next valid time
```

### The Overlap: Fall Back

```java
// In America/New_York, 2024-11-03 01:30 occurs twice
// Clocks fall back from 2:00 AM to 1:00 AM

LocalDateTime ambiguous = LocalDateTime.of(2024, 11, 3, 1, 30);
ZoneId eastern = ZoneId.of("America/New_York");

// Which 1:30 AM?
ZonedDateTime zdt = ambiguous.atZone(eastern);
// By default: the earlier occurrence (before the transition)

// To be explicit:
ZonedDateTime earlier = ambiguous.atZone(eastern)
    .withEarlierOffsetAtOverlap();
ZonedDateTime later = ambiguous.atZone(eastern)
    .withLaterOffsetAtOverlap();
```

---

## Converting from Legacy APIs

### Date ↔ Instant

```java
// Date to Instant
Date legacyDate = new Date();
Instant instant = legacyDate.toInstant();

// Instant to Date
Date backToDate = Date.from(instant);
```

### Date ↔ LocalDateTime

```java
// Date to LocalDateTime (needs timezone assumption)
Date legacyDate = new Date();
LocalDateTime ldt = legacyDate.toInstant()
    .atZone(ZoneId.systemDefault())
    .toLocalDateTime();

// LocalDateTime to Date (needs timezone assumption)
Date backToDate = Date.from(
    ldt.atZone(ZoneId.systemDefault()).toInstant()
);
```

### Calendar ↔ ZonedDateTime

```java
// Calendar to ZonedDateTime
Calendar calendar = Calendar.getInstance();
ZonedDateTime zdt = calendar.toInstant()
    .atZone(calendar.getTimeZone().toZoneId());

// ZonedDateTime to Calendar
Calendar backToCal = GregorianCalendar.from(zdt);
```

---

## Database Integration

JDBC 4.2+ supports `java.time` types directly:

```java
// Writing
PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO events (name, event_date, event_time, created_at) VALUES (?, ?, ?, ?)"
);
ps.setString(1, "Conference");
ps.setObject(2, LocalDate.of(2024, 12, 25));    // DATE column
ps.setObject(3, LocalTime.of(14, 0));            // TIME column
ps.setObject(4, Instant.now());                  // TIMESTAMP WITH TIME ZONE

// Reading
ResultSet rs = ps.executeQuery();
LocalDate date = rs.getObject("event_date", LocalDate.class);
LocalTime time = rs.getObject("event_time", LocalTime.class);
Instant created = rs.getObject("created_at", Instant.class);
```

> **Best Practice**: Store timestamps as `Instant` (or `TIMESTAMP WITH TIME ZONE` in the database). Convert to user's timezone only for display. This avoids timezone-related data corruption.

---

## Common Patterns

### Pattern 1: Age Calculation

```java
public int calculateAge(LocalDate birthDate) {
    return Period.between(birthDate, LocalDate.now()).getYears();
}
```

### Pattern 2: Business Days

```java
public LocalDate addBusinessDays(LocalDate start, int days) {
    LocalDate result = start;
    int added = 0;
    while (added < days) {
        result = result.plusDays(1);
        if (result.getDayOfWeek() != DayOfWeek.SATURDAY &&
            result.getDayOfWeek() != DayOfWeek.SUNDAY) {
            added++;
        }
    }
    return result;
}
```

### Pattern 3: Start/End of Day

```java
public Instant getStartOfDay(LocalDate date, ZoneId zone) {
    return date.atStartOfDay(zone).toInstant();
}

public Instant getEndOfDay(LocalDate date, ZoneId zone) {
    return date.atTime(LocalTime.MAX).atZone(zone).toInstant();
    // Or: date.plusDays(1).atStartOfDay(zone).toInstant().minusNanos(1);
}
```

### Pattern 4: Time Range Overlap

```java
public boolean overlaps(
    Instant start1, Instant end1,
    Instant start2, Instant end2) {

    return start1.isBefore(end2) && start2.isBefore(end1);
}
```

### Pattern 5: Scheduling in User's Timezone

```java
public Instant scheduleForUser(
    LocalDateTime userLocalTime,
    String userTimezone) {

    ZoneId zone = ZoneId.of(userTimezone);
    return userLocalTime.atZone(zone).toInstant();
}

public LocalDateTime displayToUser(
    Instant instant,
    String userTimezone) {

    ZoneId zone = ZoneId.of(userTimezone);
    return instant.atZone(zone).toLocalDateTime();
}
```

---

## What Interviewers Actually Want to Know

1. **Why is java.time better than Date/Calendar?** Immutability (thread-safe, no defensive copying), clear domain separation (LocalDate vs Instant), sensible API (January is 1, not 0).

2. **When would you use LocalDateTime vs ZonedDateTime vs Instant?**
   - `Instant`: timestamps, when events happened, database storage
   - `ZonedDateTime`: scheduling across timezones, user-facing dates with timezone
   - `LocalDateTime`: wall-clock time without timezone (rare; think "meeting at 2 PM" without specifying where)

3. **How do you handle daylight saving time?** Use `ZonedDateTime` which handles transitions automatically. Be aware of gaps (time doesn't exist) and overlaps (time exists twice).

4. **What's the difference between Duration and Period?** Duration is machine time (exact seconds/nanoseconds). Period is calendar time (years/months/days). Adding "1 month" varies by month length.

5. **How do you convert between legacy Date and java.time?** Via `Instant`: `date.toInstant()` and `Date.from(instant)`. Add timezone context for `LocalDateTime` conversion.

---

## Common Misconceptions

**"LocalDateTime represents a specific instant"**

No. `LocalDateTime.of(2024, 12, 25, 14, 0)` is "December 25, 2024 at 2 PM"—but 2 PM *where*? That's a different instant in Tokyo vs New York. For a specific instant, use `ZonedDateTime` or `Instant`.

**"Store dates in the database as strings"**

No. Use proper date/time types. String storage loses sorting, range queries, and timezone handling. JDBC 4.2+ supports `java.time` directly.

**"Timezone abbreviations like 'EST' are reliable"**

No. "EST" is ambiguous (Eastern Standard Time, not Eastern Daylight Time) and not unique globally. Use IANA timezone IDs: "America/New_York", "Europe/London", "Asia/Tokyo".

**"Midnight is always the start of the day"**

Not quite. Use `LocalDate.atStartOfDay(zone)` because some timezones skip midnight during DST transitions.

**"I can ignore timezone in my application"**

Only if all users are in the same timezone and that never changes. Otherwise, store `Instant`, convert to user timezone for display.

---

## The Philosophical Win

The old Date/Calendar API treated time as simple—just milliseconds. The `java.time` API acknowledges that time is complex:

- Human time (calendars, timezones, months of varying length)
- Machine time (precise nanoseconds on a timeline)
- Local time (what a clock shows)
- Global time (a point everyone agrees on)

By providing distinct types for these concepts, `java.time` lets the compiler catch mistakes that would be runtime bugs with Date. Passing a `LocalDate` to a method expecting an `Instant` is a compile error, not a timezone mishap discovered in production.

That's the power of a well-designed type system: making invalid states unrepresentable.

---

## Summary

| Old API | New API | Key Improvement |
|---------|---------|-----------------|
| `Date` | `Instant` | Immutable, clear name (it's an instant, not a date) |
| `Date` (as date) | `LocalDate` | No time component, no timezone confusion |
| `Calendar` | `ZonedDateTime` | Immutable, cleaner API |
| manual millis math | `Duration` | Type-safe time spans |
| manual day counting | `Period` | Calendar-aware spans |
| `SimpleDateFormat` | `DateTimeFormatter` | Immutable, thread-safe |

The `java.time` API respects the complexity of time while hiding it behind a clean interface. Use the right type for your concept, let immutability protect you from threading bugs, and store instants for timestamps.

Time is hard. Your API shouldn't make it harder.

---

*Next Chapter: [Memory and References Reimagined](05-memory-references.md)*
