
# AWK Command Examples 🐧

`awk` is a powerful text-processing tool in Linux used for **pattern scanning, filtering, and transforming** data — especially columnar data.

## Basic Syntax

```bash
awk 'pattern { action }' filename
```

## Sample Data (`employees.txt`)

```
Name        Department   Salary
Venkat      DevOps       8000
Ravi        Cloud        9000
Priya       DevOps       7500
Amit        Security     8500
Sita        Cloud        9500
```

---

## 1. Print Entire File

```bash
awk '{ print }' employees.txt
```

## 2. Print Specific Columns

```bash
awk '{ print $1, $3 }' employees.txt
```

> `$1` = first column, `$2` = second, `$3` = third, `$0` = entire line

## 3. Filter Rows by Pattern

```bash
awk '$2 == "DevOps"' employees.txt
```

## 4. Filter with Condition

```bash
# Employees with salary > 8000
awk '$3 > 8000' employees.txt
```

## 5. Print with Custom Formatting

```bash
awk '{ printf "%-10s earns $%s\n", $1, $3 }' employees.txt
```

## 6. Skip the Header (`NR > 1`)

```bash
awk 'NR > 1 { print $1, $3 }' employees.txt
```

> `NR` = **N**umber of **R**ecords (current line number)

## 7. Sum a Column

```bash
awk 'NR > 1 { sum += $3 } END { print "Total Salary:", sum }' employees.txt
```

## 8. Count Rows

```bash
awk 'NR > 1 { count++ } END { print "Total Employees:", count }' employees.txt
```

## 9. Average of a Column

```bash
awk 'NR > 1 { sum += $3; count++ } END { print "Average Salary:", sum/count }' employees.txt
```

## 10. Use Custom Delimiter (`-F`)

For a CSV file `data.csv`:

```
Venkat,DevOps,8000
Ravi,Cloud,9000
```

```bash
awk -F ',' '{ print $1, $3 }' data.csv
```

## 11. BEGIN and END Blocks

```bash
awk 'BEGIN { print "=== Employee Report ===" }
     NR > 1 { print $1, $3 }
     END { print "=== End of Report ===" }' employees.txt
```

## 12. Search for a Pattern (like `grep`)

```bash
awk '/Cloud/' employees.txt
```

## 13. Print Line Numbers

```bash
awk '{ print NR, $0 }' employees.txt
```

## 14. Replace/Modify a Field

```bash
# Give everyone a 10% raise
awk 'NR > 1 { $3 = $3 * 1.10; print }' employees.txt
```

---

## Quick Reference Cheat Sheet

| Variable/Feature | Meaning |
|---|---|
| `$0` | Entire line |
| `$1, $2, $3...` | Column 1, 2, 3... |
| `NR` | Current line number |
| `NF` | Number of fields in current line |
| `-F ','` | Set delimiter (e.g., comma) |
| `BEGIN {}` | Runs before processing starts |
| `END {}` | Runs after all lines are processed |
| `/pattern/` | Match lines containing pattern |

---

## Author

Created for learning and quick reference purposes.
