# WebApi

## 1. Updates installieren

```bash
sudo apt update
```

## 2. .NET SDK 6.0 installieren

```bash
sudo apt install dotnet-sdk-6.0
```

## 3. Projekt erstellen

```bash
mkdir <ProjektName>
cd <ProjektName>
```

```bash
dotnet new web --n <ProjektName>
```

## 4. `launchSettings.json` ändern

```json
{
    "profiles": {
        "WebApi": {
            "commandName": "Project",
            "dotnetRunMessages": true,
            "launchBrowser": true,
            "applicationUrl": "http://localhost:5001",
            "environmentVariables": {
                "ASPNETCORE_ENVIRONMENT": "Development"
            }
        }
    }
}
```

Den Inhalt der `launchSettings.json` mit dem obigen Json-Objekt ersetzen. Dies stellt sicher, dass unsere Web Applikation nur durch den Port 5001 zugängig ist.

Wenn man nun `dotnet run` im selben Verzeichnis ausführt, kann man über http://localhost:5001 auf die Seite kommen.

## 5. Multistaging

Man kann das Projekt für Multistaging so einstellen:

Neue `Dockerfile` Datei erstellen

```bash
touch Dockerfile
```

Mit folgendem ausfüllen:

```dockerfile
# 1. Build compile image
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build-env
WORKDIR /build
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o out

# 2. Build runtime image
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app
COPY --from=build-env /build/out .
ENV ASPNETCORE_URLS=http://*:5001
EXPOSE 5001
ENTRYPOINT ["dotnet", "<ProjektName>.dll"]
```

Neue `docker-compose.yml` Datei erstellen

```bash
touch docker-compose.yml
nano docker-compose.yml
```

Mit dem Nano-Editor folgendes reinschreiben:

```yaml
version: "3.9"
services:
  webapi:
    build: <ProjektName>
    ports:
     - 5001:5001
```

## 6. MongoDB einbauen

MongoDB Treiber installieren ins Projekt

```bash
dotnet add package MongoDB.Driver
```

`docker-compose.yml` Datei aktualisieren:

```yml
version: "3.9"
services:
  webapi2:
    build: WebApi2
    depends_on:
      - mongodb
    ports:
      - 5001:5001
  mongodb:
    image: mongo
    volumes:
      - mongoData:/data/db
volumes:
  mongoData:
```

`Program.cs` Quellcode aktualisieren

```cs
using MongoDB.Driver;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.MapGet("/check", () => {
   try
   {
     var mongoClient = new MongoClient("mongodb://mongodb:27017");
    var databaseNames = mongoClient.ListDatabaseNames().ToList();

    return string.Join(",\n", databaseNames);
   }
   catch (System.Exception e)
   {
    return e.Message;
   }
});

app.Run();
```

Damit der MongoDB Server läuft, muss man ihn über `docker run` starten.

Wenn man nun `dotnet run` ausführt, sollte man die Datenbanken über http://localhost:5001 sehen.