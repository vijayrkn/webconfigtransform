Sample showing the usage of Web.Config transform:

Original Web.config:
```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModule" resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath="bin\IISSupport\VSIISExeLauncher.exe" arguments="-argFile IISExeLauncherArgs.txt" stdoutLogEnabled="false" />
    </system.webServer>
  </location>
</configuration>
```

Transform files are here:
Configuration: https://github.com/vijayrkn/webconfigtransform/blob/master/Web.Release.config
Profile: https://github.com/vijayrkn/webconfigtransform/blob/master/Web.FolderProfile.config
Environment: https://github.com/vijayrkn/webconfigtransform/blob/master/Web.Staging.config
Custom : https://github.com/vijayrkn/webconfigtransform/blob/master/Custom.transform


`msbuild webconfigtransform.csproj /p:DeployOnBuild=true /p:Configuration=Release /p:PublishProfile=FolderProfile /p:EnvironmentName=Staging /p:CustomTransformFileName=custom.transform`

Running the above command will transform the web.config to below. This applies the transform at the following levels: Configuration, Profile, Environment & Custom

```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModule" resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath="dotnet" arguments=".\webconfigtransform.dll" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout">
        <environmentVariables>
          <environmentVariable name="ASPNETCORE_ENVIRONMENT" value="Staging" />
          <environmentVariable name="Configuration_Specific" value="SomeConfigurationSpecificValue"/>
          <environmentVariable name="Profile_Specific" value="SomeProfileSpecificValue"/>
          <environmentVariable name="Environment_Specific" value="SomeEnvironmentSpecificValue"/>
          <environmentVariable name="Custom" value="SomeCustomValue"/>
        </environmentVariables>
      </aspNetCore>
    </system.webServer>
  </location>
</configuration>
```


`dotnet publish /p:Configuration=Release /p:EnvironmentName=Staging /p:CustomTransformFileName=custom.transform`

Running the above command will transform the web.config to below. If the profile is not passed, then the profile specific transform is skipped.

```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModule" resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath="dotnet" arguments=".\webconfigtransform.dll" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout">
        <environmentVariables>
          <environmentVariable name="ASPNETCORE_ENVIRONMENT" value="Staging" />
          <environmentVariable name="Configuration_Specific" value="SomeConfigurationSpecificValue" />
          <environmentVariable name="Environment_Specific" value="SomeEnvironmentSpecificValue" />
          <environmentVariable name="Custom" value="SomeCustomValue" />
        </environmentVariables>
      </aspNetCore>
    </system.webServer>
  </location>
</configuration>
```

`dotnet publish /p:Configuration=Release`

Running the above command will transform the web.config to below since only configuration is passed.

```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModule" resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath="dotnet" arguments=".\webconfigtransform.dll" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" >
        <environmentVariables>
          <environmentVariable name="Configuration_Specific" value="SomeConfigurationSpecificValue"/>
        </environmentVariables>
      </aspNetCore>
    </system.webServer>
  </location>
</configuration>
```

`dotnet publish`

Running the above command will not apply any of the specified transforms since it does not satisfy any of the transform conditions. It still applies the required transforms like changing the process path etc.

```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModule" resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath="dotnet" arguments=".\webconfigtransform.dll" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" />
    </system.webServer>
  </location>
</configuration>
```


`dotnet publish /p:IsWebConfigTransformDisabled=true`

Running the above command will not apply any transforms. It will keep the original web.config

```
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModule" resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath="bin\IISSupport\VSIISExeLauncher.exe" arguments="-argFile IISExeLauncherArgs.txt" stdoutLogEnabled="false" />
    </system.webServer>
  </location>
</configuration>
```

How web.config transform works? 

Xml transforms are supported at 4 levels - Configuration, Profile, Environment & Custom.

1. Configuration specific transform : web.$(configuration).config
  - Adding a web.$(configuration).config to the root of the project.
  - This is the first transform that gets run.


2. Profile specific transform : web.`<profilename>`.config - msbuild property for profilename is $(PublishProfile)
  - Adding a web.`<profilename>`.config to the root of the project.
  - This is the second transform that gets run.
 - The profile name here is the name of the profile used. If no profile is passed, then the default profile name is 'FileSystem' (i.e web.FileSystem.config will be applied if present )


3. Environment specific transform: web.$(EnvironmentName).config
  - Adding a web.$(EnvironmentName).config to the root of the project.
  - This is the third transform that gets run.
 - EnvironmentName can be specified from the commandline or pubxml. When Environment name is specified, Web config transform will automatically add this environment variable in the web.config (ASPNETCORE_ENVIRONMENT).

4. Custom transform : <Any file name>
- Adding any transform file to the root of the project and setting the msbuild property $(CustomTransformFileName) to the name of the transform file.
- This is the last transform that gets run.

