# Technical Guide: Implementing Hijri Calendar with Intl.DateTimeFormat in JavaScript

# Hijri Calendar in JavaScript - Technical Reference

## Problem

Property applications in Saudi Arabia need to display and calculate dates in both Hijri and Gregorian calendars.

## Solution: Intl.DateTimeFormat

### Basic Display

```javascript
// Hijri (Umm al-Qura calendar)
const hijri = new Intl.DateTimeFormat('ar-SA-u-ca-islamic', {
  year: 'numeric',
  month: 'long',
  day: 'numeric'
}).format(new Date('2025-06-27'));
// Output: "1 muharram 1447"

// Gregorian
const greg = new Intl.DateTimeFormat('en-US', {
  year: 'numeric',
  month: 'long',
  day: 'numeric',
  calendar: 'gregory'
}).format(new Date('2025-06-27'));
// Output: "June 27, 2025"
```

### Dual Date Utility

```typescript
interface DualDate {
  gregorian: string;
  hijri: string;
  combined: string;
}

function formatDualDate(
  dateString: string,
  locale: 'ar' | 'en' = 'en'
): DualDate {
  const date = new Date(dateString);
  const gregLocale = locale === 'ar' ? 'ar-SA' : 'en-US';
  const hijriLocale = 'ar-SA-u-ca-islamic';
  const options: Intl.DateTimeFormatOptions = {
    year: 'numeric',
    month: 'short',
    day: 'numeric'
  };

  const gregorian = new Intl.DateTimeFormat(
    gregLocale, { ...options, calendar: 'gregory' }
  ).format(date);

  const hijri = new Intl.DateTimeFormat(
    hijriLocale, options
  ).format(date);

  return {
    gregorian,
    hijri,
    combined: `${gregorian} (${hijri})`
  };
}
```

### Prisma Data Model

```prisma
model Contract {
  id            String    @id @default(uuid())
  startDate     DateTime  // Always stored as Gregorian
  endDate       DateTime
  calendarType  CalendarType @default(GREGORIAN)
}

enum CalendarType {
  GREGORIAN
  HIJRI
}
```

## Intl Calendar Types

| Value | Calendar | Use Case |
|-------|----------|----------|
| `islamic` | General Islamic | Generic |
| `islamic-umalqura` | Umm al-Qura | Saudi Arabia (official) |
| `islamic-civil` | Civil Islamic | Mathematical |
| `islamic-tbla` | Tabular Islamic | Astronomical |

**Use `islamic-umalqura`** (or shorthand `islamic` in `ar-SA` locale) for Saudi Arabia.

## Critical Rules

1. **Always store dates as Gregorian** in PostgreSQL
2. 2. **Convert only in the presentation layer**
   3. 3. **Use Umm al-Qura calendar** for Saudi Arabia
      4. 4. **Skip external libraries** - built-in `Intl` is sufficient and more accurate
        
         5. Built at: [amlakire.com](https://amlakire.com)
