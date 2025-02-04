FROM mcr.microsoft.com/dotnet/sdk:7.0
WORKDIR /app
COPY *.csproj ./
RUN dotnet restore
COPY . ./
RUN dotnet publish -c Release -o out
RUN chmod +x out/Worker.dll
ENTRYPOINT ["dotnet", "out/Worker.dll"]