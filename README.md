<Project Sdk="Microsoft.NET.Sdk.Web">

 <PropertyGroup>
    <TargetFramework>netcoreapp3.1</TargetFramework>
   <RunAnalyzersDuringBuild>false</RunAnalyzersDuringBuild>
   <RunAnalyzersDuringLiveAnalysis>false</RunAnalyzersDuringLiveAnalysis>
 </PropertyGroup>

 <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Debug|AnyCPU'">
   <DefineConstants />
 </PropertyGroup>

 <ItemGroup>
   <Compile Remove="Controllers\WeatherForecastController.cs" />
   <Compile Remove="WeatherForecast.cs" />
 </ItemGroup>
</Project>
Severity    Code    Description    Project    File    Line    Suppression State
Error    NETSDK1045    The current .NET SDK does not support targeting .NET Core 3.1.  Either target .NET Core 2.1 or lower, or use a version of the .NET SDK that supports .NET Core 3.1.    Chat2DeskRest    C:\Program Files\dotnet\sdk\2.1.505\Sdks\Microsoft.NET.Sdk\targets\Microsoft.NET.TargetFrameworkInference.targets    137
