```Python
s = list("PAYPALISHIRING")
rows = 4
p = []
while s:
    l = [' ' for _ in range(rows)]
    for i in range(rows):
        if not s:
            break
        l[i] = s.pop(0)
    p.append(l)
    if not s:
        break
    l = [' ' for _ in range(rows)]
    for i in range(rows - 2, 0, -1):
        if not s:
            break
        l[i] = s.pop(0)
        p.append(l)
        l = [' ' for _ in range(rows)]
for i in range(rows):
    for j in p:
        print(j[i].ljust(2), end='')
    print()
```