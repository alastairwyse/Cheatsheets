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

#### Imports

For files/modules in the local project, always use the full path to the files/modules relative to the 'PYTHONPATH' environment variable...

```
from src.exceptions.element_not_found_error import ElementNotFoundError
```

#### Unit tests folder structure

Create a 'tests' folder within the package's folder

#### Check that a variable is of a specific type

And also getting the 'inner' exception ('cause') from an exception.  N.b. [CalledProcessError](https://docs.python.org/3/library/subprocess.html#subprocess.CalledProcessError) is a class in the [subprocess module](https://docs.python.org/3/library/subprocess.html)...

```python
try:
    subfolders = command_executor.execute_operating_system_command("ls -d */ -1", tar_folder_path)
except Exception as exc:
    subfolders_exist: bool = True
    if (exc.__cause__ is not None):
        if (type(exc.__cause__) == CalledProcessError):
            if (exc.__cause__.returncode == 2):
                subfolders_exist = False
```

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
        raw_results, # type: ignore[assignment]
        self._ENTITY_TYPE_JSON_NAME, 
        self._ENTITY_JSON_NAME, 
        StringUniqueStringifier(), 
        StringUniqueStringifier()
        )
    
    return results
```

[This page](https://mypy.readthedocs.io/en/stable/error_code_list.html) lists some of the Mypy-generated errors which can be referenced in the square brackets to make the declaration more specified.  For non-Mypy-generated errors, the square backets can be omitted (just use '# type: ignore').

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

Also need to use 'self' when instantiating...

```python
    stringifier = self._ApplicationScreenUniqueStringifier()
```

#### Syntax for class constants

```python
class AccessManagerClient(AccessManagerClientBase, AccessManagerEventProcessor, AccessManagerQueryProcessor, Generic[TUser, TGroup, TComponent, TAccess]):

    _USER_JSON_NAME: str = "user"
    _GROUP_JSON_NAME: str = "group"
```

As per [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html#3164-guidelines-derived-from-guidos-recommendations)

#### Enums

```python
from enum import Enum

class HTTPMethod(Enum):
    """Represents an HTTP method.
    """
    GET = "GET", 
    POST = "POST", 
    DELETE = "DELETE"
```

#### F-strings

Can be used to substitute variables into strings inline (instead of using format())...

```python
raise Exception(f"Failed to call URL '{request_url}' with '{str(HTTPMethod.GET.name)}' method.") from exc
```

#### Incrementing

Python doesn't support ++ (etc...) like other langauges.  Instead you can do this...

```python
myNumber += 1
```

#### With statement

Automatically calls close() method when finished... similar to C# 'using'...

```python
with open("file_path", "w") as file:
    file.write("hello world !")
```

#### Exception equivalent to C#

| C# Exception | Python Error/Exception | Notes |
| ------------ | ---------------------- | ----- |
| ArgumentException | ValueError | Includes C# exceptions derived from ArgumentException... ArgumentNullException, ArgumentOutOfRangeException, etc... |

#### Asserting an exception message in unit tests

```python
with self.assertRaises(ValueError) as result:
    list(self._test_json_dict_to_iterable_converter.convert_to_iterable(test_input_list, StringUniqueStringifier(), "AccessLevel")) # type: ignore

self.assertEqual("Element of parameter 'input_list' does not contain key 'AccessLevel'.", str(result.exception))
```

#### Asserting an inner exception's message in unit tests

```python
with self.assertRaises(Exception) as result:
    (statement whichs throws exception)

self.assertEqual("Inner exception message.", str(result.exception.__context__))
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
python -m unittest src.tests.access_manager_client_integration_tests.AccessManagerClientIntegrationTests.test_connection_exceptions
```

To run a single test module/file.  Specify the test file name...

```
python -m unittest src.tests.access_manager_client_integration_tests
```

#### Route execution of a test to unittest.main()

Required at the bottom of any test class...

```python
if __name__ == "__main__":
    unittest.main()
```

#### Exception handling for string being empty, None, or whitespace

```python
if (param_val is None or not param_val or param_val.isspace() == True):
    raise ValueError("Value of argument 'param_val' is None or empty.")
```

#### Mocks in unit tests

1. Import Mock and/or MagicMock (see [here](https://docs.python.org/3/library/unittest.mock.html#the-mock-class) and [here](https://docs.python.org/3/library/unittest.mock.html#magicmock-and-magic-method-support) for differences) and the 'patch' decorator...


```python
from unittest.mock import MagicMock, Mock, patch
```

2. Decorate each test method that needs to use the mock with the 'patch' decoration.  In below case the 'requests' library is being mocked.  Need to put the full path to the requests instance to mock as the parameter to 'patch'... implemented under the covers by some [tricky dynamic importing and overriding](https://docs.python.org/3/library/unittest.mock.html#patch).  That then results in the mocked object being passed as a parameter to the same test method, so need to capture that.  Also see [other variations of patch](https://docs.python.org/3/library/unittest.mock.html#the-patchers)...

```python
@patch("access_manager_client.requests")
def test_add_user_connection_error(self, mock_requests: requests)
```

3. Define the mock behaviour with the 'side-effect' parameter.  For an exception/fail case like below...

```python
mock_requests.post = MagicMock(side_effect=ConnectionError)
```

... and for a success case...

```python
mock_response = MagicMock()
mock_response.status_code = 201
mock_response.text = "ok"
mock_requests.post.return_value = mock_response
```

#### Basic Loops

C# 'foreach' equivalent...

```python
for current_user_entities in self._user_to_entity_map.values():
    if entity_type in current_user_entities:
        current_user_entities.pop(entity_type)
```

...subject of the iteration must implement [__iter__()](https://docs.python.org/3/library/functions.html#iter)

Can use the range() function for more traditional for loops...

```python
for i in range(0, 10):
    print(i)
```

#### Getter and Setter Equivalents (@property)

```python
class _CityAndTemperature():

    _city: str
    _temperature: int

    def __init__(self, city: str, temperature: int) -> None:
        
        # TODO: Checks for blank/empty city

        self._city = city
        self._temperature = temperature

    @property
    def city(self) -> str:
        return self._city
    
    
    @property
    def temperature(self) -> int:
        return self._temperature
    
    @temperature.setter
    def temperature(self, new_temperature: int) -> None:
        self._temperature = new_temperature
```


TODO: Asserting mocks were called with certain parameters and specified numbers of times.
