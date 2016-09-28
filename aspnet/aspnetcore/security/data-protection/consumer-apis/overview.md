﻿---
uid: security/data-protection/consumer-apis/overview
---
# Consumer APIs Overview

The IDataProtectionProvider and IDataProtector interfaces are the basic interfaces through which consumers use the data protection system. They are located in the Microsoft.AspNetCore.DataProtection.Interfaces package.

## IDataProtectionProvider

The provider interface represents the root of the data protection system. It cannot directly be used to protect or unprotect data. Instead, the consumer must get a reference to an IDataProtector by calling IDataProtectionProvider.CreateProtector(purpose), where purpose is a string that describes the intended consumer use case. See [Purpose Strings](purpose-strings.md) for much more information on the intent of this parameter and how to choose an appropriate value.

## IDataProtector

The protector interface is returned by a call to CreateProtector, and it is this interface which consumers can use to perform protect and unprotect operations.

To protect a piece of data, pass the data to the Protect method. The basic interface defines a method which converts byte[] -> byte[], but there is also an overload (provided as an extension method) which converts string -> string. The security offered by the two methods is identical; the developer should choose whichever overload is most convenient for his use case. Irrespective of the overload chosen, the value returned by the Protect method is now protected (enciphered and tamper-proofed), and the application can send it to an untrusted client.

To unprotect a previously-protected piece of data, pass the protected data to the Unprotect method. (There are byte[]-based and string-based overloads for developer convenience.) If the protected payload was generated by an earlier call to Protect on this same IDataProtector, the Unprotect method will return the original unprotected payload. If the protected payload has been tampered with or was produced by a different IDataProtector, the Unprotect method will throw CryptographicException.

The concept of same vs. different IDataProtector ties back to the concept of purpose. If two IDataProtector instances were generated from the same root IDataProtectionProvider but via different purpose strings in the call to IDataProtectionProvider.CreateProtector, then they are considered [different protectors](purpose-strings.md), and one will not be able to unprotect payloads generated by the other.

## Consuming these interfaces

For a DI-aware component, the intended usage is that the component take an IDataProtectionProvider parameter in its constructor and that the DI system automatically provides this service when the component is instantiated.

> [!NOTE]
> Some applications (such as console applications or ASP.NET 4.x applications) might not be DI-aware so cannot use the mechanism described here. For these scenarios consult the [Non DI Aware Scenarios](../configuration/non-di-scenarios.md) document for more information on getting an instance of an IDataProtection provider without going through DI.

The following sample demonstrates three concepts:

1. [Adding the data protection system](../configuration/overview.md) to the service container,

2. Using DI to receive an instance of an IDataProtectionProvider, and

3. Creating an IDataProtector from an IDataProtectionProvider and using it to protect and unprotect data.

[!code-none[Main](../using-data-protection/samples/protectunprotect.cs?highlight=26,34,35,36,37,38,39,40)]

````none

   using System;
   using Microsoft.AspNetCore.DataProtection;
   using Microsoft.Extensions.DependencyInjection;

   public class Program
   {
       public static void Main(string[] args)
       {
           // add data protection services
           var serviceCollection = new ServiceCollection();
           serviceCollection.AddDataProtection();
           var services = serviceCollection.BuildServiceProvider();

           // create an instance of MyClass using the service provider
           var instance = ActivatorUtilities.CreateInstance<MyClass>(services);
           instance.RunSample();
       }

       public class MyClass
       {
           IDataProtector _protector;

           // the 'provider' parameter is provided by DI
           public MyClass(IDataProtectionProvider provider)
           {
               _protector = provider.CreateProtector("Contoso.MyClass.v1");
           }

           public void RunSample()
           {
               Console.Write("Enter input: ");
               string input = Console.ReadLine();

               // protect the payload
               string protectedPayload = _protector.Protect(input);
               Console.WriteLine($"Protect returned: {protectedPayload}");

               // unprotect the payload
               string unprotectedPayload = _protector.Unprotect(protectedPayload);
               Console.WriteLine($"Unprotect returned: {unprotectedPayload}");
           }
       }
   }

   /*
    * SAMPLE OUTPUT
    *
    * Enter input: Hello world!
    * Protect returned: CfDJ8ICcgQwZZhlAlTZT...OdfH66i1PnGmpCR5e441xQ
    * Unprotect returned: Hello world!
    */

   ````

The package Microsoft.AspNetCore.DataProtection.Abstractions contains an extension method IServiceProvider.GetDataProtector as a developer convenience. It encapsulates as a single operation both retrieving an IDataProtectionProvider from the service provider and calling IDataProtectionProvider.CreateProtector. The following sample demonstrates its usage.

[!code-none[Main](./overview/samples/getdataprotector.cs?highlight=15)]

````none

   using System;
   using Microsoft.AspNetCore.DataProtection;
   using Microsoft.Extensions.DependencyInjection;
    
   public class Program
   {
       public static void Main(string[] args)
       {
           // add data protection services
           var serviceCollection = new ServiceCollection();
           serviceCollection.AddDataProtection();
           var services = serviceCollection.BuildServiceProvider();
    
           // get an IDataProtector from the IServiceProvider
           var protector = services.GetDataProtector("Contoso.Example.v2");
           Console.Write("Enter input: ");
           string input = Console.ReadLine();
    
           // protect the payload
           string protectedPayload = protector.Protect(input);
           Console.WriteLine($"Protect returned: {protectedPayload}");
    
           // unprotect the payload
           string unprotectedPayload = protector.Unprotect(protectedPayload);
           Console.WriteLine($"Unprotect returned: {unprotectedPayload}");
       }
   }

   ````

>[!TIP]
> Instances of IDataProtectionProvider and IDataProtector are thread-safe for multiple callers. It is intended that once a component gets a reference to an IDataProtector via a call to CreateProtector, it will use that reference for multiple calls to Protect and Unprotect.A call to Unprotect will throw CryptographicException if the protected payload cannot be verified or deciphered. Some components may wish to ignore errors during unprotect operations; a component which reads authentication cookies might handle this error and treat the request as if it had no cookie at all rather than fail the request outright. Components which want this behavior should specifically catch CryptographicException instead of swallowing all exceptions.