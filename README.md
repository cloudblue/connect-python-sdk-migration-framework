# Connect Migration Middleware

Small middleware to ease the service migration from legacy to Connect.

## Installation

If you have a requirements file in your project, just add the following line to add the repository as a dependency:

```
-e git+https://github.com/ingrammicro/connect-python-sdk-migration-framework#egg=connect-migration
```

You can also install it directly passing the previous statement to `pip install` if you don't rely on a requirements file for some reason.

If you don't use `pip` at all, simply copy the `connect_migration.py` file from the repository to your project.

## Usage

This package provides the `MigrationHandler` class, which helps migrating data from a legacy service into Connect. The class initializer has the following parameters:

| Parameter         | Type                 | Description |
| ----------------- | -------------------- | ----------- |
| `transformations` | `Dict[str,callable]` | Contains the param id as keys, and the function that produces the value of the parameter. |
| `migration_key`   | `str`                | The name of the Connect parameter that stores the legacy data in JSON format. Default value is `migration_info`. |
| `serialize`       | `bool`               | If `True`, it will automatically serialize any non-string value in the migration data on direct assignation flow. Default value is `False`. |

The functions passed to the `transformations` array will receive two arguments:

| Parameter        | Type             | Description |
| ---------------- | ---------------- | ----------- |
| `transform_data` | `Dict[str, Any]` | Contains the entire transformation information as a JSON object, parsed from the param with `migration_key` id. |
| `request_id`     | `str`            | The id of the request being processed. |

We can use an instance of this class into our request processor, like this:

```python
from connect.models import ActivationTileResponse
from connect.resources import FulfillmentAutomation

from connect_migration import MigrationHandler


class ProductFulfillment(FulfillmentAutomation):
    def __init__(self):
        super(ProductFulfillment, self).__init__()
        
        # Create a migration handler and define transformation functions
        self.migration_handler = MigrationHandler({
            'email': lambda data, request_id: data['teamAdminEmail'].upper(),
            'team_id': lambda data, request_id: data['teamId'].upper(),
            'team_name': lambda data, request_id: data['teamName'].upper(),
            'num_licensed_users': lambda data, request_id: int(data['licNumber']) * 10
        })

    def process_request(self, request):
        if request.type == 'purchase':
            if request.needs_migration():
                # The migrate() method returns a new request object with
                # the parameter values updated, we must update the parameters
                # and approve the fulfillment
                request = self.migration_handler.migrate(request)
                self.update_parameters(request.id, request.asset.params)
                return ActivationTileResponse('The data has been migrated :)')
            else:
                # We perform normal processing here ...
                pass
```

The previous example converts string migration data values to uppercase, while it multiplies the integer value of parameter `licNumber` by 10.

## Exceptions

The package defines two exception classes:

| Exception                 | Description |
| ------------------------- | ----------- |
| `MigrationParamError`     | Can be thrown if any parameter fails on validation and/or transformation, an error will be logged for that parameter and the migration will fail. The fulfillment should be skipped. |
| `MigrationAbortError`     | The migration will directly fail, the fulfillment should be skipped. |
