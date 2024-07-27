#### Calling base class constructor for Exceptions

```python
def __init__(self, *args: Tuple) -> None:
    super().__init__(*args)
```

#### Documenting a package

Each folder should have an '__init.py__' file with a comment describing the package contents, e.g...

```python
"""Container/model classes used by ApplicationAccess AccessManager client classes.

"""
```

#### Asserting an inner exception's message in unit tests

```python
with self.assertRaises(Exception) as result:
    (statement whichs throws exception)

self.assertEqual("Inner exception message.", str(result.exception.__context__))
```

#### Unit tests folder structure

Create a 'tests' folder within the package's folder

#### Mocking in unit tests

TODO
