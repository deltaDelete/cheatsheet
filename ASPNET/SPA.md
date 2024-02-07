# Single-page application

This guide is based of Visual Studio's React + ASP.NET Template

## Prerequisites
1. Installed NodeJS
2. Existing or newly created ASP.NET 7+ project (CSPROJ in separate directory of SLN)

Directory that contains SLN file will be referenced as `ROOT`

Directory that contains ASP.NET CSPROJ file will be referenced as `SERVER`

Directory with React will be referenced as `CLIENT`

## Steps

### 1. Open ROOT directory in terminal

### 2. Create React subproject

#### npm
```shell
npm create vite@latest my-app.client --template react-ts
```

#### Yarn
```shell
yarn create vite my-app.client --template react-ts
```

### 3. Create an ESPROJ file inside `CLIENT`

```xml
<!-- my-app.client.esproj -->
<Project Sdk="Microsoft.VisualStudio.JavaScript.Sdk/0.5.128-alpha">
    <PropertyGroup>
        <StartupCommand>npm run dev</StartupCommand>
        <!-- or if you use yarn-->
        <!-- <StartupCommand>yarn dev</StartupCommand>-->
        <JavaScriptTestFramework>Jest</JavaScriptTestFramework>
        <!-- Allows the build (or compile) script located on package.json to run on Build -->
        <ShouldRunBuildScript>false</ShouldRunBuildScript>
        <!-- Folder where production build objects will be placed -->
        <PublishAssetsDirectory>$(MSBuildProjectDirectory)\dist</PublishAssetsDirectory>
        <!-- Without this dotnet-cli doesn't want to add project -->
        <ProjectTypeGuids>54A90642-561A-4BB1-A94E-469ADEE60C69</ProjectTypeGuids>
    </PropertyGroup>
</Project>
```

### 4. Add ESPROJ to SLN

Use IDE or this command

```shell
dotnet sln add ROOT/my-app.client/my-app.client.esproj
```

### 5. Add Some properties to `SERVER`.csproj

Pay attention to `$PORT$`, that should be replaced by your Vite port

```xml

<Project Sdk="Microsoft.NET.Sdk.Web">
    
    <PropertyGroup>
        ...
        <SpaRoot>..\my-app.client</SpaRoot>
        <SpaProxyLaunchCommand>npm run dev</SpaProxyLaunchCommand>
        <SpaProxyLaunchCommand>yarn dev</SpaProxyLaunchCommand>
        <SpaProxyServerUrl>https://localhost:$PORT$</SpaProxyServerUrl>
    </PropertyGroup>
    
    <ItemGroup>
        ...
        <PackageReference Include="Microsoft.AspNetCore.SpaProxy">
            <Version>8.*-*</Version>
        </PackageReference>
    </ItemGroup>
    
    <ItemGroup>
        <ProjectReference Include="..\reactapp1.client\reactapp1.client.esproj">
            <ReferenceOutputAssembly>false</ReferenceOutputAssembly>
        </ProjectReference>
    </ItemGroup>
    
</Project>
```

### 6. Configure Vite

vite.config.ts created by Visual Studio
```ts
import { fileURLToPath, URL } from 'node:url';

import { defineConfig } from 'vite';
import plugin from '@vitejs/plugin-react';
import fs from 'fs';
import path from 'path';
import child_process from 'child_process';

const baseFolder =
    process.env.APPDATA !== undefined && process.env.APPDATA !== ''
        ? `${process.env.APPDATA}/ASP.NET/https`
        : `${process.env.HOME}/.aspnet/https`;

const certificateArg = process.argv.map(arg => arg.match(/--name=(?<value>.+)/i)).filter(Boolean)[0];
const certificateName = certificateArg ? certificateArg.groups.value : "my-app.client";

if (!certificateName) {
    console.error('Invalid certificate name. Run this script in the context of an npm/yarn script or pass --name=<<app>> explicitly.')
    process.exit(-1);
}

const certFilePath = path.join(baseFolder, `${certificateName}.pem`);
const keyFilePath = path.join(baseFolder, `${certificateName}.key`);

if (!fs.existsSync(certFilePath) || !fs.existsSync(keyFilePath)) {
    if (0 !== child_process.spawnSync('dotnet', [
        'dev-certs',
        'https',
        '--export-path',
        certFilePath,
        '--format',
        'Pem',
        '--no-password',
    ], { stdio: 'inherit', }).status) {
        throw new Error("Could not create certificate.");
    }
}

// https://vitejs.dev/config/
export default defineConfig({
    plugins: [plugin()],
    resolve: {
        alias: {
            '@': fileURLToPath(new URL('./src', import.meta.url))
        }
    },
    server: {
        proxy: {
            '^/weatherforecast': {
                target: 'https://localhost:7186/',
                secure: false
            }
        },
        port: 5173,
        https: {
            key: fs.readFileSync(keyFilePath),
            cert: fs.readFileSync(certFilePath),
        }
    }
})
```

As you can see this config specifies ASP.NET's HTTPS certificate path and in case if it is not found creates it.

Also it specifies endpoints that will be proxified to `SERVER`

### 7. Edit `SERVER`/Properties/launchSettings.json

For each profile you must add following entry
```json
"ASPNETCORE_HOSTINGSTARTUPASSEMBLIES": "Microsoft.AspNetCore.SpaProxy"
```

Like this:
```json
{
  "$schema": "http://json.schemastore.org/launchsettings.json",
  "iisSettings": {
    "windowsAuthentication": false,
    "anonymousAuthentication": true,
    "iisExpress": {
      "applicationUrl": "http://localhost:44642",
      "sslPort": 44366
    }
  },
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "http://localhost:5169",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development",
        "ASPNETCORE_HOSTINGSTARTUPASSEMBLIES": "Microsoft.AspNetCore.SpaProxy"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "https://localhost:7186;http://localhost:5169",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development",
        "ASPNETCORE_HOSTINGSTARTUPASSEMBLIES": "Microsoft.AspNetCore.SpaProxy"
      }
    },
    "IIS Express": {
      "commandName": "IISExpress",
      "launchBrowser": true,
      "launchUrl": "swagger",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development",
        "ASPNETCORE_HOSTINGSTARTUPASSEMBLIES": "Microsoft.AspNetCore.SpaProxy"
      }
    }
  }
}
```

Also make sure port specified in Vite is the same as in launchSettings.json

### 8. Launch your project

```shell
dotnet run --project `SERVER`.csproj
```

If everything is made perfectly ASP.NET server should start and then start Vite with React application

### Example of React component which interacts with `SERVER`

```tsx
import {useEffect, useState} from 'react';
import './App.css';

function App() {
    const [forecasts, setForecasts] = useState<Forecast[]>();

    useEffect(() => {
        populateWeatherData();
    }, []);

    const contents = forecasts === undefined
        ? <p><em>Loading... Please refresh once the ASP.NET backend has started. See <a
            href="https://aka.ms/jspsintegrationreact">https://aka.ms/jspsintegrationreact</a> for more details.</em>
        </p>
        : <table className="table table-striped" aria-labelledby="tabelLabel">
            <thead>
            <tr>
                <th>Date</th>
                <th>Temp. (C)</th>
                <th>Temp. (F)</th>
                <th>Summary</th>
            </tr>
            </thead>
            <tbody>
            {forecasts.map(forecast =>
                <tr key={forecast.date}>
                    <td>{forecast.date}</td>
                    <td>{forecast.temperatureC}</td>
                    <td>{forecast.temperatureF}</td>
                    <td>{forecast.summary}</td>
                </tr>
            )}
            </tbody>
        </table>;

    return (
        <div>
            <h1 id="tabelLabel">Weather forecast</h1>
            <p>This component demonstrates fetching data from the server.</p>
            {contents}
        </div>
    );

    async function populateWeatherData() {
        const response = await fetch('weatherforecast');
        const data = await response.json();
        setForecasts(data);
    }
}

export default App;
```