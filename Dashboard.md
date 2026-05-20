# CTF Dashboard

## Tất cả Writeups

```dataview
TABLE platform, category, difficulty, points, tags, solved
FROM "01-Writeups"
SORT date DESC
```

---

## Forensics

```dataview
TABLE platform, date, points, tags
FROM "01-Writeups"
WHERE category = "Forensics" AND solved = true
SORT points DESC
```

---

## Web

```dataview
TABLE platform, date, points, tags
FROM "01-Writeups"
WHERE category = "Web" AND solved = true
SORT points DESC
```

---

## Crypto

```dataview
TABLE platform, date, points, tags
FROM "01-Writeups"
WHERE category = "Crypto" AND solved = true
SORT points DESC
```

---

## Pwn

```dataview
TABLE platform, date, points, tags
FROM "01-Writeups"
WHERE category = "Pwn" AND solved = true
SORT points DESC
```

---

## Rev

```dataview
TABLE platform, date, points, tags
FROM "01-Writeups"
WHERE category = "Rev" AND solved = true
SORT points DESC
```

---

## Thống kê

```dataview
TABLE length(rows) AS "Số bài"
FROM "01-Writeups"
WHERE solved = true
GROUP BY category
```