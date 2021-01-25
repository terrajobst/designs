# Multi-targeting for RIDs

**PM** [Immo Landwerth](https://github.com/terrajobst)

## Notes

* You can't differ package references by RID. However, we can rely on linker to
  remove the dependency. It would just mean that the package author should
  exclude their dependency from being added to the dependency. We *could* do
  conditional package references, but that only works partially (i.e. the NuGet
  package will always depend on the package, no matter the RID), but the compile
  time items would.
* Sync with Claire on how `MSBuild.Extras` works
* Should we support reference assemblies? Eric thinks no for packages because it
  can result in "TFM holes" where one can resolve a ref but not an impl
* There is a public item to add files to a particular location in the package
* One hard problem is P2P, because we want the same semantics in P2P that the
  resulting package gets.
    - Ideally this doesn't depend on the default packaging behavior and would
      just inspect a well-known item and it's NuGetFolderPath and apply regular
      NuGet rules.
* For native assets you want two different ways of building:
    - The final package, which is a multi-machine build
    - A specific vertical, which is the machine's RID
* We need to think about how multi OS builds are done
    - Ideally 
* The SQLite scenario *would not be* multi-targeted because it's a single
  managed binary with multiple native assets, which are static data.
* Look at
    - `System.IO.Ports`
    - `Microsoft.Data.SqlClient`
* We need validation rules
    - Like, if you multi-target with RIDs, you must have a target without a RID.
* Non-goal:
    - RID-specific dependencies or partial package support
* `<None Pack="True" PackagePath="lib\" />`
    - Note packaged by default
* `<Content PackagePath="lib\" />`
    - Packaged by default
* `<TfmSpecificPackageFile />`
* https://docs.microsoft.com/en-us/nuget/reference/msbuild-targets
* https://github.com/NuGet/NuGet.Client/blob/b6a116e8356d913074f2a2051990cf834119376e/src/NuGet.Core/NuGet.Build.Tasks.Pack/NuGet.Build.Tasks.Pack.targets#L491-L534

## Scenarios and User Experience

### Different dependencies for Blazor WebAssembly

Finley is building an enterprise portal using Blazor WebAssembly. As part of the
portal, they have to read and write payloads for SAP. Some of these require MD5
checksums.

Since crypto APIs aren't supported in WebAssembly, they decide to take a
dependency on the popular NuGet library `Contoso.ManagedMD5` that offers a
managed implementation of MD5 that will also work in WebAssembly.

However, Finely also wants to share this component with their server side code
where they can use the `System.Security.Cryptography.MD5` just fine. So they
decide to multi-target between .NET 5.0 and .NET 5.0 with the `browser` RID.
This allows them to add a conditional package reference:

```XML
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>net5.0;net5.0#browser</TargetFrameworks>
  </PropertyGroup>

  <ItemGroup Condition="'$(TargetFramework)' == 'net5.0#browser'">
    <PackageReference Include="Contoso.ManagedMD5" />
  </ItemGroup>

</Project>
```

In code, they can use conditional compilation to special case the `browser` RID:

```C#
private string ComputeMd5(byte[] input)
{
    byte[] bytes;
#if BROWSER
    bytes = Contoso.ManagedMD5.HashData(input);
#else
    bytes = System.Security.Cryptography.MD5.HashData(input);
#endif
    return Convert.ToHexString(bytes);
}
```

### Invoking and distributing native code

Eric is providing the SQLite package. Since SQLite is a native component, he
needs to offer different implementations, based on operating system and
architecture.

```XML
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>
        netstandard1.0;
        netstandard1.0#linux-x64;
        netstandard1.0#osx-x64;
        netstandard1.0#win7-x86;
        netstandard1.0#win7-x64;
    </TargetFrameworks>
  </PropertyGroup>

  <ItemGroup Condition="'$(RuntimeIdentifier)' == 'linux-x64'">
    <RuntimeAsset Include="$(NativeFiles)linux/libsqlite3.so" />
  </ItemGroup>

  <ItemGroup Condition="'$(RuntimeIdentifier)' == 'osx-x64'">
    <RuntimeAsset Include="$(NativeFiles)/osx/libsqlite3.dylib" />
  </ItemGroup>

  <ItemGroup Condition="'$(RuntimeIdentifier)' == 'win-x86'">
    <RuntimeAsset Include="$(NativeFiles)/win7/x86/sqlite3.dll" />
  </ItemGroup>

  <ItemGroup Condition="'$(RuntimeIdentifier)' == 'win-x64'">
    <RuntimeAsset Include="$(NativeFiles)/win7/x64/sqlite3.dll" />
  </ItemGroup>

</Project>
```

### Supporting platform & architecture neutral deployments

When building for `netstandard1.0` without a RID, we want the output folder to
contain the native assets for all operating systems & architectures. This allows
for XCOPY deployments for applications that are otherwise not setting a RID.

TODO: What's the pattern for folks to use different paths to the native EXE?
      Jan seems to indicate that the builder/loader handles that already.
      See: `Mono.Posix.NETStandard`

TODO: Do we want to building packages where `runtime` and `lib` can be disjoint,
      i.e. a single `lib\netstandard1.0\my.dll` while having multiple
      `runtime\<rid>\native\foo.xxx` entries. Like compare `Mono.Posix.NETStandard`
      with `SQLite`.

## Requirements

### Goals

### Non-Goals

## Design

## Q & A
