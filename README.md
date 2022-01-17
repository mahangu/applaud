# Applaud

`Applaud` is a Python client library for accessing [App Store Connect API](https://developer.apple.com/documentation/appstoreconnectapi), generated by [Applaudgen](https://github.com/codinn/applaudgen).

## Features

- [x] Support App Store Connect API latest version 1.6
- [x] Support `filter`, `fileds`, `include`, `limit`, `sort`, `exists` and other query parameters
- [x] All endpoints / paths are implemented, include, but not limited to: App Information, TestFlight, Users and Roles, Sales and Finances
- [x] Pythonic, all `camelCase` schema fields are represented as `snake_case` class attributes
- [x] Embrace [Python type hints](https://www.python.org/dev/peps/pep-0483/)
- [x] Use [Python Requests](https://docs.python-requests.org/en/latest/) to hanlde HTTP sessions
- [x] [ErrorResponse](https://developer.apple.com/documentation/appstoreconnectapi/errorresponse) can be catched as exception

## Installation

Install with `pip`:

```
pip install applaud
```

Install with [Poetry](https://python-poetry.org/):
```
poetry add applaud
```

Install with [Pipenv](https://pipenv.pypa.io/en/latest/):
```
pipenv install applaud
```

## Usage

Calls to the API require authorization, so before we get started, you obtain keys to create the tokens from your organization’s App Store Connect account. See [Creating API Keys for App Store Connect API](https://developer.apple.com/documentation/appstoreconnectapi/creating_api_keys_for_app_store_connect_api) to create your keys and tokens.


### Connection

`Connection` is the core class of `Applaud`, it holds a connection between client and remote service, [generate a new token](https://developer.apple.com/documentation/appstoreconnectapi/generating_tokens_for_api_requests#3878467) before it expires.

```python
from applaud.connection import Connection

# Create a connection object using API keys
connection = Connection(APPSTORE_ISSUER_ID, APPSTORE_KEY_ID, APPSTORE_PRIVATE_KEY)
```

In most of cases, all tasks you'd like to perform on remote service should be initiated from a `Connection` object. `Connection` has a bunch of functions help you create `…Endpoint` objects:

```python
# Return an AppListEndpoint object
connection.apps()
```

### Endpoint

A `…Endpoint` class encapsulates all operations you can perform on a specific resource. For example, this snippet fetches first two (sort by app name) apps that "ready for sale" and have game center enabled versions:

```python
# Return an AppsResponse object
connection.apps().filter(
    app_store_versions_app_store_state=AppStoreVersionState.READY_FOR_SALE
).exists(
    game_center_enabled_versions=True
).limit(
    2
).sort(
    name: SortOrder.ASC
).get()
```

#### `…Endpoint.get()`

The `get()` operation initiates a HTTP `GET` request on the endpoint's path. For example, the URL of [List Apps](https://developer.apple.com/documentation/appstoreconnectapi/list_apps) service endpoint is:
```
GET https://api.appstoreconnect.apple.com/v1/apps
```

The corresponding code in `Applaud`:
```python
# Return an AppsResponse object
response = connection.apps().get()

for app in response.data:
    print(f'{app.attributes.name}: {app.attributes.bundle_id}')
```

Unlike other operations (`create()`, `update()` and `delete()`), `get()` operation can be chained by query parameters functions:

**`filter()`**

You use `filter()` function to extract matching resources. For example:
```
filter[bundleId]  Attributes, relationships, and IDs by which to filter.
        [string]
```

The corresponding code in `Applaud`:
```python
response = connection.apps().filter(
    bundle_id="com.exmaple.app1"
).get()
# or
connection.apps().filter(
    bundle_id=["com.exmaple.app1", "com.exmaple.app2"]
).get()

for app in response.data:
    print(f'{app.attributes.name}: {app.attributes.bundle_id}')
```

**`include()`**

You use `include()` function to ask relationship data to include in the response. For example:
```
 include  Relationship data to include in the response.
[string]  Possible values: appClips, appInfos, appStoreVersions,
          availableTerritories, betaAppLocalizations,
          betaAppReviewDetail, betaGroups, betaLicenseAgreement,
          builds, ciProduct, endUserLicenseAgreement,
          gameCenterEnabledVersions, inAppPurchases, preOrder,
          preReleaseVersions, prices
```

The corresponding code in `Applaud`:
```python
response = connection.apps().include(
    AppListEndpoint.Include.BETA_LICENSE_AGREEMENT
).get()
# or
response = connection.apps().include(
    [AppListEndpoint.Include.BETA_LICENSE_AGREEMENT, AppListEndpoint.Include.PRICES]
).get()
```

**`fields()`**

You use `fields()` function to ask fields data to return for included related resources in a `get()` operation. Related resources specified in `fields()` function *MUST* be included explicitly in `include()` function, otherwise, the remote service may not return the fields data that you expect. For example:
```
fields[betaLicenseAgreements]  Fields to return for included related types.
                     [string]  Possible values: agreementText, app
```

The corresponding code in `Applaud`:
```python
connection.apps().include(
    AppListEndpoint.Include.BETA_LICENSE_AGREEMENT
).fields(
    beta_license_agreement=[BetaLicenseAgreementField.AGREEMENT_TEXT, BetaLicenseAgreementField.APP]
).get()
```

**`limit()`**

You use `limit()` function to restrict the maximum number of resources to return in a `get()` operation. For example:
```
  limit  Number of resources to return.
integer  Maximum Value: 200
```

The corresponding code in `Applaud`:
```python
# Return a response contains 10 apps at most
connection.apps().limit(10).get()

# Raise a ValueError exception, the maxinmu allowed value is 200
connection.apps().limit(400).get()
```

You can also included limit the number of related resources to return, as in `fields()` function, you *MUST* also specify the related resources explicitly in `include()` function. For example:
```
limit[appStoreVersions]  integer
                         Maximum Value: 50
```

The corresponding code in `Applaud`:
```python
# All returned apps have 5 related app store version at most
connection.apps().include(
    AppListEndpoint.Include.APP_STORE_VERSIONS
).limit(app_store_versions=5).get()

# Raise a ValueError exception, the maxinmu allowed value is 50
connection.apps().include(
    AppListEndpoint.Include.APP_STORE_VERSIONS
).limit(app_store_versions=100).get()
```

By leverage `limit()` function with `sort()` function, your script can be more responsive.

**`sort()`**

You use `sort()` function to sort the returned resources by attributes in ascending or descending order. For example:
```
    sort  Attributes by which to sort.
[string]  Possible values: bundleId, -bundleId, name, -name, sku, -sku
```

The corresponding code in `Applaud`:
```python
connection.apps().sort(name=SortOrder.ASC, bundleId=SortOrder.DESC).get()
```

**`exists()`**

`exists()` is a special type of filter – filter by existence or non-existence of related resource. For example:
```
exists[gameCenterEnabledVersions]  [string]
```

The corresponding code in `Applaud`:
```python
connection.apps().exists(game_center_enabled_versions=True).get()
```

#### `…Endpoint.create()`

The `create()` operation initiates a HTTP `POST` request on the endpoint's path. For example, the URL of [Create an App Store Version](https://developer.apple.com/documentation/appstoreconnectapi/create_an_app_store_version) service endpoint is:
```
POST https://api.appstoreconnect.apple.com/v1/appStoreVersions
```

The corresponding code in `Applaud`:
```python
request = AppStoreVersionCreateRequest(
            data = AppStoreVersionCreateRequest.Data(
                relationships = AppStoreVersionCreateRequest.Data.Relationships(
                    app = AppStoreVersionCreateRequest.Data.Relationships.App(
                        data = AppStoreVersionCreateRequest.Data.Relationships.App.Data(
                            id = 'com.exmaple.app1'
                        )
                    )
                ),
                attributes = AppStoreVersionCreateRequest.Data.Attributes(
                    version_string = '1.6',
                    platform = Platform.IOS,
                    copyright = f'Copyright © 2021 Codinn Technologies. All rights reserved.',
                    release_type = AppStoreVersionReleaseType.AFTER_APPROVAL
                )
            )
        )

# Return an AppStoreVersionResponse object
reponse = connection.app_store_versions().create(request)

version = response.data
print(f'{version.version_string}: {version.created_date}, {version.app_store_state}')
```

#### `…Endpoint.update()`

The `update()` operation initiates a HTTP `PATCH` request on the endpoint's path. For example, the URL of [Modify an App Store Version](https://developer.apple.com/documentation/appstoreconnectapi/modify_an_app_store_version) service endpoint is:
```
PATCH https://api.appstoreconnect.apple.com/v1/appStoreVersions/{id}
```

The corresponding code in `Applaud`:
```python
# Get the version id created in previous example
version_id = version.data.id

# Update version's information
request = AppStoreVersionUpdateRequest(
            data = AppStoreVersionUpdateRequest.Data(
                id = version.data.id,

                attributes = AppStoreVersionUpdateRequest.Data.Attributes(
                    version_string = '1.6.1',
                    platform = Platform.IOS,
                    copyright = f'Copyright © 2022 Codinn Technologies. All rights reserved.',
                    release_type = AppStoreVersionReleaseType.AFTER_APPROVAL
                )
            )
        )

# Return an AppStoreVersionResponse object
reponse = connection.app_store_version(version_id).update(request)

version = response.data
print(f'{version.version_string}: {version.copyright}, {version.app_store_state}')
```

#### `…Endpoint.delete()`

The `delete()` operation initiates a HTTP `DELETE` request on the endpoint's path. For example, the URL of [Delete an App Store Version](https://developer.apple.com/documentation/appstoreconnectapi/delete_an_app_store_version) service endpoint is:
```
DELETE https://api.appstoreconnect.apple.com/v1/appStoreVersions/{id}
```

The corresponding code in `Applaud`:
```python
# Get the version id created in previous example
version_id = version.data.id

connection.app_store_version(version_id).delete()
```

### Exceptions

`…Endpoint.get()`, `…Endpoint.create()`, `…Endpoint.update()` and `…Endpoint.delete()` may raise two types of exceptions:

1. HTTP request exceptions raised by [Python Requests](https://docs.python-requests.org/en/latest/api/#exceptions)
2. Remote service returns an [ErrorResponse](https://developer.apple.com/documentation/appstoreconnectapi/errorresponse)

For the second case, `Applaud` raises an `EndpointException` exception, and attaches all [`ErrorResponse.Error`](https://developer.apple.com/documentation/appstoreconnectapi/errorresponse/errors) objects in `EndpointException.errors` attribute.

Some errors are harmless, for example, App Store Connect has no API for you to tell whether a tester has accepted beta test invitation or not. When you trying to resend invitations to testers, you may encounter an `ALREADY_ACCEPTED` error, it's safe to just ignore such error:

```python
try:
    # Send / resend beta test invitations
    response = connection.beta_tester_invitations().create(…)
except EndpointException as err:
    already_accepted_error = False
    for e in err.errors:
        if e.code == 'STATE_ERROR.TESTER_INVITE.ALREADY_ACCEPTED':
            # silent this error
            already_accepted_error = True
            break

    if not already_accepted_error:
        raise err
```

## Caveats

- Query parameters functions (`filter()`, `include`, `fields` …) play a role only when using with `…Endpoint.get()`. Though there is no side effects if chain it with `…Endpoint.create()`, `…Endpoint.update()` and `…Endpoint.delete()` operations, it is not adviced.

