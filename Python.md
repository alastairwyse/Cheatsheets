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

#### Preventing false positive errors for type hints

'assert' statement...

```python
@property
def entity_types(self) -> Iterable[str]:
    url: str = self._base_url + "entityTypes"
    raw_results = self._send_get_request(url)
    assert isinstance(raw_results, List)
    
    return raw_results
```

'type: ignore' to switch off type checking for a single line.  In below case _send_get_request() returns a JSON-like Dict defined as Union[str, List[str], Dict[str, Any]].  We know that 'raw_results' will be a List[Dict[str, Any]] as required by method _json_to_iterable_converter(), but Pylance does not...

```python
def get_user_to_entity_mappings(self, user: TUser) -> Iterable[Tuple[str, str]]:
    url: str = self._base_url + "userToEntityMappings/user/{0}?includeIndirectMappings=false".format(
        self._encode_url_component(self._user_stringifier.to_string(user))
    )
    raw_results = self._send_get_request(url)
    results: Iterable[Tuple[str, str]] = self._json_to_iterable_converter.convert_to_iterable_of_tuples(
        raw_results, # type: ignore
        self._ENTITY_TYPE_JSON_NAME, 
        self._ENTITY_JSON_NAME, 
        StringUniqueStringifier(), 
        StringUniqueStringifier()
        )
    
    return results
```

Really good reference for these in [this post](https://stackoverflow.com/questions/68446642/how-do-i-get-pylance-to-ignore-the-possibility-of-none).

#### Raising an exception from another

```python
try:
    response: Response = requests.request(
        str(HTTPMethod.GET.name), 
        request_url, 
        headers=self._headers, 
        auth=self._auth, 
        timeout=self._timeout, 
        proxies=self._proxies, 
        verify=self._verify, 
        cert=self._cert
    )
except Exception as exc:
    raise Exception("Failed to call URL '{0}' with '{1}' method.".format(request_url, str(HTTPMethod.GET.name))) from exc
```

#### Inner classes

Class definition should be indented within another class, and follow the convention for private members (underscore prefix in name)...

```python
    class _ApplicationScreenUniqueStringifier(UniqueStringifierBase[ApplicationScreen]):

        def to_string(self, input_object: ApplicationScreen) -> str:
            if (input_object == ApplicationScreen.order):
                return "Order"
            elif (input_object == ApplicationScreen.summary):
                return "Summary"
            elif (input_object == ApplicationScreen.manage_products):
                return "ManageProducts"
            elif (input_object == ApplicationScreen.settings):
                return "Settings"
            elif (input_object == ApplicationScreen.delivery):
                return "Delivery"
            elif (input_object == ApplicationScreen.review):
                return "Review"
            else:
                raise ValueError("Unhandled ApplicationScreen value '{0}'.".format(input_object))
```

#### Exception equivalent to C#

| C# Exception | Python Erro/Exception | Notes |
| ------------ | --------------------- | ----- |
| ArgumentException | ValueError | Includes C# exceptions derived from ArgumentException... ArgumentNullException, ArgumentOutOfRangeException, etc... |

#### Asserting an exception message in unit tests

```python
with self.assertRaises(ValueError) as result:
    list(self._test_json_dict_to_iterable_converter.convert_to_iterable(test_input_list, StringUniqueStringifier(), "AccessLevel")) # type: ignore

self.assertEqual("Element of parameter 'input_list' does not contain key 'AccessLevel'.", str(result.exception))
```

#### Running unit tests

To run all tests in a project.  Details of parameters...

| Parameter | Description |
| --------- | ----------- |
| -m | Directory to start test discovery |
| -p | Pattern to match test files |

```
python -m unittest discover -s .\src\tests -p "*_tests.py"
```

To run a single test.  Specify the test file name and the test method name (including the class/module name prefix)...

```
python src\tests\access_manager_client_integration_tests.py AccessManagerClientIntegrationTests.test_add_elements_and_mappings
```

#### Route execution of a test to unittest.main()

Required at the bottom of any test class...

```python
if __name__ == "__main__":
    unittest.main()
```

TODO
