# Sample project

This repository is a sample related to [NuGet/Home#10578](https://github.com/NuGet/Home/issues/10578) and [NuGet/Home#6764](https://github.com/NuGet/Home/issues/6764)

## Description

As described in the issue mentioned above, we have a C# project (`ProjectA`). This project depends on a project (`ProjectB`) as well as a package (`Package1`) that depends on another package (`Package2`) not mentioned in the project file: 

```
ProjectA <- ProjectB
    ^
    +------ Package1 <- Package2

// (where A <- B means that A depends on B).
```

We are trying to apply custom metadata to `Package1` resolution, say `ExcludeAssets=runtime`. The status quo is to apply these metadata to all its references, transitively.

## Behavior

Using the metadata mentioned above, `ExcludeAssets=runtime` applied on `Package1`, we expect `Package2` resolution to also exclude `runtime` assets; Or at least this is currently the case when not using centrally managed package versions.

To showcase this, in the repository `Package1` is `System.Reflection.Emit.Lightweight` and `Package2` is `System.Reflection.Emit.ILGeneration`. When restoring without using centrally managed packages versions, we have the expected and current behavior:

ProjectA.cspoj
```xml
<PackageReference Include="System.Reflection.Emit.Lightweight" Version="4.7.0" ExcludeAssets="runtime" />
```

Command
```
dotnet restore
```

project.assets.json (`ProjectA`)
```json
"System.Reflection.Emit.ILGeneration/4.7.0": {
  "type": "package",
  "compile": {
    "ref/netstandard2.0/System.Reflection.Emit.ILGeneration.dll": {}
  },
  "runtime": {
    "lib/netstandard2.0/_._": {}
  }
},
"System.Reflection.Emit.Lightweight/4.7.0": {
  "type": "package",
  "dependencies": {
    "System.Reflection.Emit.ILGeneration": "4.7.0"
  },
  "compile": {
    "ref/netstandard2.0/System.Reflection.Emit.Lightweight.dll": {}
  },
  "runtime": {
    "lib/netstandard2.0/_._": {}
  }
}
```

In comparison, the behavior with centrally managed package versions is:

Command
```
dotnet restore -p:ManagePackageVersionsCentrally=true
```

project.assets.json (`ProjectA`)
```diff
"System.Reflection.Emit.ILGeneration/4.7.0": {
  "type": "package",
  "compile": {
    "ref/netstandard2.0/System.Reflection.Emit.ILGeneration.dll": {}
  },
  "runtime": {
-   "lib/netstandard2.0/_._": {}
+   "lib/netstandard2.0/System.Reflection.Emit.ILGeneration.dll": {}
  }
},
"System.Reflection.Emit.Lightweight/4.7.0": {
  "type": "package",
  "dependencies": {
    "System.Reflection.Emit.ILGeneration": "4.7.0"
  },
  "compile": {
    "ref/netstandard2.0/System.Reflection.Emit.Lightweight.dll": {}
  },
  "runtime": {
    "lib/netstandard2.0/_._": {}
  }
}
```

It appears that this is due to the fact that, when using centrally managed package versions, `System.Reflection.Emit.Lightweight` is wrongly mentioned as a dependency of `ProjectB` (therefore its dependencies are transitively resolved) even if `ProjectB` does not depend on any package:

project.assets.json (`ProjectA`)
```diff
"ProjectB/1.0.0": {
  "type": "project",
  "framework": ".NETStandard,Version=v2.0",
+ "dependencies": {
+   "System.Reflection.Emit.Lightweight": "4.7.0"
+ },
  "compile": {
    "bin/placeholder/ProjectB.dll": {}
  },
  "runtime": {
    "bin/placeholder/ProjectB.dll": {}
  }
}
```
