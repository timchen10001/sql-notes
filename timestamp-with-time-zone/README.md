## Checkout ```pg_timezone_names```

```sql
SELECT * FROM pg_catalog.pg_timezone_names
```

## Postgres 有兩種 timestamp data type

1. ```timestamp with time zone``` = timestamptz

2. ```timestamp without time zone``` = timestamp

## Source Code

[timestamp.h](https://doxygen.postgresql.org/datatype_2timestamp_8h_source.html)

```
28 * Timestamps, as well as the h/m/s fields of intervals, are stored as
29 * int64 values with units of microseconds.  (Once upon a time they were
30 * double values with units of seconds.)
```

## Internal storage and epoch

- timestamps occupy ```8 bytes``` of storage on disk and in RAM.

- It is an Integer value representing the count of microseconds from Postgres epoch, ```2000-01-01 00:00:00 UTC```.

# timestamp

- No Time Zone is provided explicitly, Postgres *ignore* any time zone, modifier added to the input literal by mistake.

- No hours are shift for display, *`With everything happening in the same zone this is fine. For a different time zone the meaning changed !!! but value and display stay the same !!!`*

# timestamptz

- For *timestamp with time zone*, the internally stored value is ```always in UTC```

- The ```time zone itself is never stored```, which is stored ```-``` or output decorator used to compute the local time for display ```-``` with appended time zone `offset`.

- if you don't append an `offset` for `timestamptz` on input, the current time zone setting of the session is assumed. `ALL COMPUTATIONS are done with UTC timestamp values.`

> All timezone-aware dates and times are stored internally in UTC. They are converted to local time in the zone specified by the [TimeZone](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-TIMEZONE) configuration parameter before being displayed to the client.

```sql
SELECT
      typname,
      typlen
FROM
      pg_type
WHERE
      typname ~'^timestamp';


   typname   | typlen
-------------+--------
 timestamp   |      8
 timestamptz |      8
(2 rows)
```

> It’s important to note that `timestamptz` value is stored as a `UTC value`. PostgreSQL does not store any timezone data with the timestamptz value.

## PostgreSQL timestamp example

---

> check database server timestamp
```sql
SHOW TIMEZONE

      TimeZone
---------------------
 Asia/Taipei
(1 row)
```

> set timezone manually

```sql
-- list all alternatives
SELECT NAME FROM pg_catalog.pg_timezone_names;

-- set timezone manually
SET timezone = 'Asia/Taipei';
```

> create a table that consists of both timestamp the timestamptz columns.

```sql
CREATE TABLE timestamp_demo (
    ts TIMESTAMP, 
    tstz TIMESTAMPTZ
);
```

```sql
INSERT INTO timestamp_demo
(ts, tstz)
VALUES
('2022-01-13 09:53:00-08','2022-01-13 09:53:00-08')
RETURNING
ts, tstz;

         ts          |          tstz
---------------------+------------------------
2022-01-13 09:53:00  | 2022-01-14 01:53:00 +0800
(1 row)
```

> change the time zone of the current session to `America/New_York` and query data again.

```sql
SET timezone = 'America/New_York';

SELECT ts, tstz
FROM timestamp_demo;

         ts          |          tstz
---------------------+------------------------
2022-01-13 09:53:00  | 2022-01-14 01:53:00 +0800
(1 row)

```

# WHY ?

## In summary, it allows:
- TIMESTAMP WITHOUT  TIME ZONE ⇾ TIMESTAMP WITH  TIME ZONE (add time zone)
- TIMESTAMP WITH  TIME ZONE ⇾ TIMESTAMP WITHOUT  TIME ZONE (shift time zone)

## #1 AT TIME ZONE adding time zone designations:
```sql
SELECT '2018-09-02 07:09:19'::timestamp AT TIME ZONE 'America/Chicago';

        timezone

------------------------

 2018-09-02 08:09:19-04


SELECT '2018-09-02 07:09:19'::timestamp AT TIME ZONE 'America/Los_Angeles';

        timezone

------------------------

 2018-09-02 10:09:19-04


SELECT '2018-09-02 07:09:19'::timestamp AT TIME ZONE 'Asia/Tokyo';

        timezone

------------------------

 2018-09-01 18:09:19-04

```

> What is basically happening above is that `the date and time are interpreted as being in the specified time zone` (e.g., America/Chicago), a TIMESTAMP WITH  TIME ZONE value is created, and the value displayed in the default time zone (-04). `It doesn't matter if a time zone designation is specified in the ::timestamp` string — only the date and time are used.

---

### This is because casting a value to *TIMESTAMP WITHOUT  TIME ZONE* ignores any specified time zone:

```sql
SELECT '2018-09-02 07:09:19'::timestamp;

        timezone

------------------------

 2018-09-02 07:09:19-04


SELECT '2018-09-02 07:09:19-10'::timestamp;

        timezone

------------------------

 2018-09-02 07:09:19-04


SELECT '2018-09-02 07:09:19-12'::timestamp;

        timezone

------------------------

 2018-09-02 07:09:19-04

```

---

### This behavior is also shown in *AT TIME ZONE*:

```sql
SELECT '2018-09-02 07:09:19'::timestamp AT TIME ZONE 'America/Chicago';

        timezone

------------------------

 2018-09-02 08:09:19-04


SELECT '2018-09-02 07:09:19-10'::timestamp AT TIME ZONE 'America/Chicago';

        timezone

------------------------

 2018-09-02 08:09:19-04


SELECT '2018-09-02 07:09:19-12'::timestamp AT TIME ZONE 'America/Chicago';

        timezone

------------------------

 2018-09-02 08:09:19-04
```

---

## #2 The second use of `AT TIME ZONE` is for removing time zone designations by shifting the `TIMESTAMP WITH  TIME ZONE` value to a different time zone and removing the time zone designation:

```sql
SELECT '2018-09-02 07:09:19-04'::timestamptz AT TIME ZONE 'America/Chicago';

      timezone

---------------------

 2018-09-02 06:09:19


SELECT '2018-09-02 07:09:19-04'::timestamptz AT TIME ZONE 'America/Los_Angeles';

      timezone

---------------------

 2018-09-02 04:09:19


SELECT '2018-09-02 07:09:19-04'::timestamptz AT TIME ZONE 'Asia/Tokyo';

      timezone

---------------------

 2018-09-02 20:09:19
```

---

### In these cases, because the inputs are `TIMESTAMP WITH  TIME ZONE`, time zone designations in the strings are significant:

```sql
SELECT '2018-09-02 07:09:19-04'::timestamptz AT TIME ZONE 'America/Chicago';

      timezone

---------------------

 2018-09-02 06:09:19


SELECT '2018-09-02 07:09:19-05'::timestamptz AT TIME ZONE 'America/Chicago';

      timezone

---------------------

 2018-09-02 07:09:19


SELECT '2018-09-02 07:09:19-06'::timestamptz AT TIME ZONE 'America/Chicago';

      timezone

---------------------

 2018-09-02 08:09:19
 
```

> The time zone is not being added to the date and time. Rather, the full date/time/time zone value is shifted to the desired time zone `(America/Chicago)`, and the time zone designation removed `(TIMESTAMP WITHOUT  TIME ZONE)`. This is useful because normally you would need to change your TimeZone setting to see values in other time zones.

---

### Without the cast, `AT TIME ZONE` inputs are assumed to be `TIMESTAMP WITH  TIME ZONE`, and the local time zone is assumed if not specified:

```sql
SELECT '2018-09-02 07:09:19' AT TIME ZONE 'America/Chicago';

      timezone

---------------------

 2018-09-02 06:09:19


SELECT '2018-09-02 07:09:19-10' AT TIME ZONE 'America/Chicago';

      timezone

---------------------

 2018-09-02 12:09:19
```

---

### The most interesting queries are these two, though they return the same output as input:

```sql
SELECT '2018-09-02 07:09:19'::timestamp AT TIME ZONE 'America/Chicago' AT TIME ZONE 'America/Chicago';

      timezone

---------------------

 2018-09-02 07:09:19


SELECT '2018-09-02 07:09:19-04'::timestamptz AT TIME ZONE 'America/Chicago' AT TIME ZONE 'America/Chicago';

        timezone

------------------------

 2018-09-02 07:09:19-04
```

> As you can see the two `AT TIME ZONE` calls cancel each other out. The first creates a `TIMESTAMP WITH  TIME ZONE` in the `America/Chicago` time zone using the supplied date and time, and then shifts the value to that same time zone, removing the time zone designation.

> The second creates a `TIMESTAMP WITHOUT  TIME ZONE` value in the same time zone, then creates a `TIMESTAMP WITH  TIME ZONE` value using the date and time in the default time zone `(TimeZone)`.

---

## Using different time zones for the two calls yields useful results:

```sql
SELECT '2018-09-02 07:09:19'::timestamp AT TIME ZONE 'Asia/Tokyo' AT TIME ZONE 'America/Chicago';

      timezone

---------------------

 2018-09-01 17:09:19
```

> This gives the `America/Chicago` time for the supplied `Asia/Tokyo` time