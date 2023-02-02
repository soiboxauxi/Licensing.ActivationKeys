# Licensing library from SNBSLibs

Free. Easy to use. No license files. **Everything is XML-documented in the code, so IntelliSense can show you all info about any member.**

If you need to know about all aspects of using this library, or you want to test it, consider the branch `tests` containing the unit tests created for this library.

## Examples of usage

### Using `LicensingClient`

Say, we have an app that isn't completely free, and we want to sell licenses for it and activate it through activation keys.

1. First, we need a database to store licenses. This library supports MS SQL Server and MySQL. You may use any database hosting from Windows Azure to [FreeSQLDatabase](https://freesqldatabase.com) (uses MySQL 5.0.12). **Not an advertisement**. Get a connection string and go to the next step.

2. Then we need to start a `LicensingClient` in the main method. It will decide whether to run the full app version or a message "Not licensed".

```c#
using SNBS.ActivationKeys;

// ...

public static void Main(string[] args) {
  LicensingClient.Start(
     "YourConnectionString", "YourProductName", false, null,
     client => {
       // Start the full version
     },
     (client, usability) => {
       // What you do when the app isn't licensed
     }
  );
}
```

3. Let's analyze this code. Method `Start` is static. In the first parameter, it takes the connection string to your database. In the second parameter you pass the name of your project — it is used to store the license information in the registry (licenses for different products store in different places).

4. The third parameter is of type `bool`. It specifies whether the `LicensingClient` should try to connect to MySQL (if it's `false`, the client will try to connect to MS SQL Server). If it's `true`, you should also set the fourth parameter (of type `Version?`) to the version of MySQL. If MS SQL Server is used, this parameter should be `null`. If the third parameter is `true`, but the fourth one is `null`, an `ArgumentException` is thrown.

5. The third parameter has type `Action<LicensingClient>` and is ran when your product has a valid license. The `LicensingClient` instance passed to it can be used to fetch the license, reactivate/deactivate your product and validate activation keys (without using them).

6. The fourth parameter has type `Action<LicensingClient, LicenseUsability>` and is ran where there's no license or an invalid license (configured in the registry for the current product). The `LicensingClient` passed can be used for the same things as described in paragraph 4. **`LicenseUsability` is an enumeration describing reasons why a license is usable/not usable.** Its values are: `Usable`, `Expired`, `NotFound`, `TooManyDevices` (each license can be used by a limited number of devices, set when it was created) and `NoConfiguredLicense`. They should be intuitive. (The difference between `NotFound` and `NoConfiguredLicense` — `NotFound` means a license is configured, but it doesn't exist in the license database. `NoConfiguredLicense` means there's *no license at all*.)

This was the most common usage of the library, but there are other ways, e.g. you can create a `LicensingClient` yourself (specify connection string, product name, use MySQL or not and the version of MySQL, as in the previous example):

```c#
var client = new LicensingClient("YourConnectionString", "YourProductName", false, null);
var usability = client.GetCurrentLicense().Usability;

if (usability != LicenseUsability.Usable) {
  ShowMessage("Your license " +
    (usability == LicenseUsability.Expired) ? "has expired" :
    (usability == LicenseUsability.NotFound) ? "was canceled" :
    (usability == LicenseUsability.NoConfiguredLicense) ? "configuration was corrupted" :
    "was corrupted");
}
```

When you create a `LicensingClient` using the constructor, it automatically connects to the licenses database. Method `GetCurrentLicense()` retrieves the currently used activation key (stored in the registry) and looks up in the database to verify it. The returned type is **structure `LicenseInfo`**. It should be obvious that it contains detailed information about one license. Its properties are:

 - `Key` of type `string`;
 - `Expiration` of type `DateTime` (only date is stored, `DateTime` instead of `DateOnly` was used because of the Entity Framework's mapping mechanism);
 - `Type` of type `LicenseType` (enumeration containing values `Trial`, `General`, `Professional`);
 - `Usability` of type `LicenseUsability`.
 
#### Applying activation keys
 
Let's improve the previous example. Generally, applications should ask the end user to activate them if the current license is not usable. The corresponding method of `LicensingClient` is called `ActivateProduct()`. It returns `LicenseInfo` containing the information about the newly activated license (of course, it's activated only if it's usable).
 
 ```c#
var client = new LicensingClient("YourConnectionString", "YourProductName");
var usability = client.GetCurrentLicense().Usability;

if (usability != LicenseUsability.Usable) {
  ShowMessage("Your license " +
    (usability == LicenseUsability.Expired) ? "has expired" :
    (usability == LicenseUsability.NotFound) ? "was canceled" :
    (usability == LicenseUsability.NoConfiguredLicense) ? "configuration was corrupted" :
    "was corrupted");
    
  string key = AskUser("Enter an activation key");
  var info = client.ActivateProduct(key);
  
  if (info.Usability == LicenseUsability.Usable) {
    ShowMessage("License successfully activated! Expires at " + info.Expiration.ToShortDateString());
  } else {
    ShowMessage("An error occurred when trying to activate. The license" +
      (usability == LicenseUsability.Expired) ? "has expired" :
      (usability == LicenseUsability.NotFound) ? "was canceled" :
      (usability == LicenseUsability.NoConfiguredLicense) ? "configuration was corrupted" :
      "was corrupted");
  }
}
```
 
A less common method is `ValidateLicense(string key)` which retrieves `LicenseInfo` of the license with the passed key, but doesn't try to activate it on the current device.

