---
title: T4 Text Transform with the dotnet CLI
description: >-
  
released: '2025-09-03T11:07:25.370Z'
cover: images/cover.jpg
author: Lukas Dürrenberger
tags:
  - C#
  - T4
  - Code Generation
shortDescription: >-
  For T4 scripts dotnet's own MSBuild version is bad news, as the necessary
  MSBuild targets and tooling only ship with Visual Studio
---

The `dotnet` command line interface (CLI) has been the recommended way to build, test, pack, publish, etc. .NET applications ever since its release. The big advantage is, that the tool is cross-platform and doesn't depend on Visual Studio. One downside is, that when you run `dotnet build` on Windows, the `dotnet` command will use its own version and environment of MSBuild to build your application, which may not provide the full power of Visual Studio that you're used to. For [T4 (Text Template Transformation Toolkit)](https://www.hanselman.com/blog/t4-text-template-transformation-toolkit-code-generation-best-kept-visual-studio-secret) scripts this is bad news, as the necessary MSBuild targets and tooling only ship with Visual Studio or [Visual Studio Build Tools](https://aka.ms/vs/17/release/vs_BuildTools.exe).

If you still want to use T4 scripts, you're left with a few options:

- Use Visual Studio's `msbuild` in your build command
- Only transform files locally with Visual Studio
- Switch to [Mono.TextTemplating](https://github.com/mono/t4), requiring some changes to the T4 scripts
- Switch to “modern” solutions like [Source Generators](https://devblogs.microsoft.com/dotnet/introducing-c-source-generators/)
- Manually call `TextTransform.exe`

If you‘re using Windows and Visual Studio Build Tools anyways, then the first or last options might be quite appealing. However, getting the last option to work during build time, requires a bit of additional engineering.

[![Transforming T4 templates using TextTransform.exe...
TextTransform.exe path: C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\Common7\IDE\TextTransform.exe
Found 16 T4 template(s)](images/build-message.png)](images/build-message.png)

## Custom TextTemplating.Targets

Here‘s a complete `*.targets` file that will work within Visual Studio, JetBrains Rider, and with the `dotnet` CLI – as long as you are on Windows and have Visual Studio 2022 or its Build Tools installed.

A brief explanation of the script:

- When run from within Visual Studio, we can directly use the `Microsoft.TextTemplating.Targets`
- Otherwise we “emulate” said targets file by providing some matching MSBuild targets and properties
- The `TransformAll` target does the heavy lifting
    - First we try to find all `*.tt` files for the given project
    - Before we can call `TextTransform.exe`, we need to determine, where it‘s located by checking potential Visual Studio installation paths
    - We log some info on what is being transformed
    - Then call `TestTransform.exe` which transforms all the collected files
- Finally the `TransformDuringBuild` target automatically triggers the transformation when building and not just when explicitly calling the `TransformAll` target

```xml
<?xml version="1.0" encoding="utf-8" ?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
   
  <!-- Try to import Visual Studio's TextTemplating targets first -->
  <Import Project="$(MSBuildExtensionsPath)\Microsoft\VisualStudio\v$(VisualStudioVersion)\TextTemplating\Microsoft.TextTemplating.targets"
          Condition="Exists('$(MSBuildExtensionsPath)\Microsoft\VisualStudio\v$(VisualStudioVersion)\TextTemplating\Microsoft.TextTemplating.targets')" />
  <!-- Rider, dotnet CLI, or similar non-VS environments: Define our own T4 transformation -->
  <PropertyGroup>
    <!-- Check if Visual Studio's T4 targets are available -->
    <HasVisualStudioT4Targets>true</HasVisualStudioT4Targets>
    <HasVisualStudioT4Targets Condition="!Exists('$(MSBuildExtensionsPath)\Microsoft\VisualStudio\v$(VisualStudioVersion)\TextTemplating\Microsoft.TextTemplating.targets')">false</HasVisualStudioT4Targets>
     
    <!-- Default behavior: TransformOnBuild is disabled -->
    <TransformOnBuild Condition="'$(TransformOnBuild)' == '' And '$(HasVisualStudioT4Targets)' == 'false'">false</TransformOnBuild>
     
    <!-- Empty default BeforeTransform and AfterTransform targets -->
    <BeforeTransform Condition="'$(BeforeTransform)' == '' And '$(HasVisualStudioT4Targets)' == 'false'"></BeforeTransform>
    <AfterTransform Condition="'$(AfterTransform)' == '' And '$(HasVisualStudioT4Targets)' == 'false'"></AfterTransform>
  </PropertyGroup>
  <!-- Custom T4 transformation target, only runs during build time, and can optionally be called directly -->
  <Target Name="TransformAll"
          DependsOnTargets="$(BeforeTransform)"
          Condition="'$(HasVisualStudioT4Targets)' == 'false'">
     
    <!-- Collect all T4 scripts of the given target -->
    <ItemGroup>
      <T4Templates Include="**/*.tt" />
    </ItemGroup>
    <!-- Find TextTransform.exe for different VS2022 editions and installation paths -->
    <PropertyGroup>
      <TextTransformPath Condition="'$(TextTransformPath)' == '' And '$(ProgramFiles)' != '' And Exists('$(ProgramFiles)\Microsoft Visual Studio\2022\BuildTools\Common7\IDE\TextTransform.exe')">$(ProgramFiles)\Microsoft Visual Studio\2022\BuildTools\Common7\IDE\TextTransform.exe</TextTransformPath>
      <TextTransformPath Condition="'$(TextTransformPath)' == '' And '$(ProgramFiles)' != '' And Exists('$(ProgramFiles)\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\TextTransform.exe')">$(ProgramFiles)\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\TextTransform.exe</TextTransformPath>
      <TextTransformPath Condition="'$(TextTransformPath)' == '' And '$(ProgramFiles)' != '' And Exists('$(ProgramFiles)\Microsoft Visual Studio\2022\Professional\Common7\IDE\TextTransform.exe')">$(ProgramFiles)\Microsoft Visual Studio\2022\Professional\Common7\IDE\TextTransform.exe</TextTransformPath>
      <TextTransformPath Condition="'$(TextTransformPath)' == '' And '$(ProgramFiles)' != '' And Exists('$(ProgramFiles)\Microsoft Visual Studio\2022\Community\Common7\IDE\TextTransform.exe')">$(ProgramFiles)\Microsoft Visual Studio\2022\Community\Common7\IDE\TextTransform.exe</TextTransformPath>
      <TextTransformPath Condition="'$(TextTransformPath)' == '' And '$(MSBuildProgramFiles32)' != '' And Exists('$(MSBuildProgramFiles32)\Microsoft Visual Studio\2022\BuildTools\Common7\IDE\TextTransform.exe')">$(MSBuildProgramFiles32)\Microsoft Visual Studio\2022\BuildTools\Common7\IDE\TextTransform.exe</TextTransformPath>
      <TextTransformPath Condition="'$(TextTransformPath)' == '' And '$(MSBuildProgramFiles32)' != '' And Exists('$(MSBuildProgramFiles32)\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\TextTransform.exe')">$(MSBuildProgramFiles32)\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\TextTransform.exe</TextTransformPath>
      <TextTransformPath Condition="'$(TextTransformPath)' == '' And '$(MSBuildProgramFiles32)' != '' And Exists('$(MSBuildProgramFiles32)\Microsoft Visual Studio\2022\Professional\Common7\IDE\TextTransform.exe')">$(MSBuildProgramFiles32)\Microsoft Visual Studio\2022\Professional\Common7\IDE\TextTransform.exe</TextTransformPath>
      <TextTransformPath Condition="'$(TextTransformPath)' == '' And '$(MSBuildProgramFiles32)' != '' And Exists('$(MSBuildProgramFiles32)\Microsoft Visual Studio\2022\Community\Common7\IDE\TextTransform.exe')">$(MSBuildProgramFiles32)\Microsoft Visual Studio\2022\Community\Common7\IDE\TextTransform.exe</TextTransformPath>
    </PropertyGroup>
     
    <Message Text="Transforming T4 templates using TextTransform.exe..." Importance="high" />
    <Message Text="TextTransform.exe path: $(TextTransformPath)" Importance="high" />
    <Message Text="Found @(T4Templates->Count()) T4 template(s)" Importance="high" />
     
    <Error Condition="@(T4Templates) != '' And !Exists('$(TextTransformPath)')"
           Text="TextTransform.exe not found. Please ensure Visual Studio 2022 is installed or manually set the TextTransformPath property." />
    <!-- Execute the transformation, using no -out argument places it at the same location as the script with the extension specified in the script -->
    <Exec Command="&quot;$(TextTransformPath)&quot; &quot;%(T4Templates.FullPath)&quot;"
        ContinueOnError="false"
        Condition="@(T4Templates) != '' And Exists('$(TextTransformPath)')"
        IgnoreExitCode="false" />
    <!-- Call AfterTransform targets if specified -->
    <CallTarget Targets="$(AfterTransform)" Condition="'$(AfterTransform)' != ''" />
  </Target>
  <!-- Target that runs during build when TransformOnBuild is true -->
  <Target Name="TransformDuringBuild"
          DependsOnTargets="TransformAll"
          BeforeTargets="BeforeBuild"
          Condition="'$(TransformOnBuild)' == 'true' And '$(HasVisualStudioT4Targets)' == 'false'" />
</Project>
```

## Usage

To make use of this target, add the following lines to your `*.csproj` file.

```xml
<Import Project="TextTemplating.targets" />
<PropertyGroup>
  <TransformOnBuild>true</TransformOnBuild>
</PropertyGroup>
```

You have the following customization options:

- `TransformOnBuild`: This enables (`true`) or disables (`false`) the transformation on build
- `BeforeTransform`: This allows you to run other MSBuild targets before the transformation
- `AfterTransform`: This allows you to run other MSBuild targets after the transformation

## Limitations

As pointed out previously, there are a few limitations:

- Only works on Windows
- Requires Visual Studio or Visual Studio Build Tools
- Assumes standard install paths – though this can easily be customized
- Doesn't implement the full feature set of `Microsoft.TextTransform.targets`
- Runs the transformation on every build, i.e. there's no change tracking
- All `*.tt` files are transformed
- The output location is the same as the T4 script location

Hope this helps someone to get their T4 scripts to transform again in their CI solution.