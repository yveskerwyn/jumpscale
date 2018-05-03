# SAL Examples

```python
path = j.sal.fs.joinPaths('/', 'tmp', 'foo')
j.sal.fs.writeFile(path, 'hello world!')

j.sal.fs.readFile(path)
'hello world!'
```