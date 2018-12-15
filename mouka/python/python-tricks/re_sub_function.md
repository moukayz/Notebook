# re.sub使用替换函数

## re.sub\(pattern, repl, string, count=0, flags=0 \)

## Use function as repl

```python
>>> text = 'UPPER PYTHON, lower python, Mixed Python'
>>> re.findall('python', text, flags=re.IGNORECASE)
['PYTHON', 'python', 'Python']
>>> re.sub('python', 'snake', text, flags=re.IGNORECASE)
'UPPER snake, lower snake, Mixed snake'
```

function will receive a match object as input parameter

```python
def replace(m):
        word = 'snake'
        text = m.group()
        if text.isupper():
            return word.upper()
        elif text.islower():
            return word.lower()
        elif text[0].isupper():
            return word.capitalize()
        else:
            return word

>>> re.sub('python', replace, text, flags=re.IGNORECASE)
'UPPER SNAKE, lower snake, Mixed Snake'
```

