# BUILD STAGE
FROM --platform=${BUILDPLATFORM} mcr.microsoft.com/dotnet/sdk:7.0 AS build

ARG TARGETPLATFORM
ARG TARGETARCH
ARG BUILDPLATFORM

WORKDIR /source

COPY . .

RUN echo "TARGETPLATFORM=$TARGETPLATFORM, TARGETARCH=$TARGETARCH, BUILDPLATFORM=$BUILDPLATFORM" &&\
    dotnet restore -a $TARGETARCH &&\
    dotnet publish -c release -o /app -a $TARGETARCH --self-contained false --no-restore

# FINAL STAGE
FROM --platform=${BUILDPLATFORM} mcr.microsoft.com/dotnet/runtime:7.0

WORKDIR /app

COPY --from=build /app .

ENTRYPOINT [ "dotnet", "Worker.dll" ]