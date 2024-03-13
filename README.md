# Chained Graurd
This repo contains an npm base image built with apko and melange:

- Package source code into APKs using [`melange`](https://github.com/chainguard-dev/melange)
- Build and publish OCI images from APKs using [`apko`](https://github.com/chainguard-dev/apko)

The app itself is a basic HTTP server that returns "Hello World!"

```
$ curl -s http://localhost:8080
Hello World!
```

### Build apks with melange

Make sure the `packages/` directory is removed:
```
rm -rf ./packages/
```

Create a temporary melange keypair:
```
docker run --rm -v "${PWD}":/work distroless.dev/melange keygen
```

Build an apk for all architectures using melange:
```
docker run --rm --privileged -v "${PWD}":/work \
    distroless.dev/melange build melange.yaml \
    --arch amd64,aarch64,armv7 \
    --repository-append packages --signing-key melange.rsa
```



### Build image with apko

*Note: you could skip this step and go to "Push image with apko".*

Build an apk for all architectures using melange:
```
docker run --rm -v "${PWD}":/work distroless.dev/apko build --debug apko.yaml "${IMAGE_NAME}" output.tar -k melange.rsa.pub  --build-arch "${ARCH}
```

If you do not wish to push the image, you could load it directly:
```
docker load < output.tar
docker run --rm --rm -p 8080:8080  "${REF}"
```

To debug the above:
```
docker run --rm -it -v "${PWD}":/work -e REF="${REF}" --entrypoint sh distroless.dev/apko

# Build image (use just --build-arch amd64 to isolate issue)
apko build --debug apko.yaml "${REF}" output.tar -k melange.rsa.pub --build-arch amd64,aarch64,armv7
```

## Push image with apko

Build and push an image to, for example, GHCR:
```
# Your GitHub username
GITHUB_USERNAME="myuser"
REF="ghcr.io/${GITHUB_USERNAME}/hello-melange-apko/$(basename "${PWD}")"

# A personal access token with the "write:packages" scope
GITHUB_TOKEN="*****"

docker run --rm -v "${PWD}":/work \
    -e REF="${REF}" \
    -e GITHUB_USERNAME="${GITHUB_USERNAME}" \
    -e GITHUB_TOKEN="${GITHUB_TOKEN}" \
    --entrypoint sh \
    distroless.dev/apko -c \
        'echo "${GITHUB_TOKEN}" | \
            apko login ghcr.io -u "${GITHUB_USERNAME}" --password-stdin && \
            apko publish --debug apko.yaml \
                "${REF}" -k melange.rsa.pub \
                --arch amd64,aarch64,armv7'
```

## cosign

After the image has been published, ['sign'](https://docs.sigstore.dev/cosign/sign#keyless-signing) and ['verify'](https://docs.sigstore.dev/cosign/verify) using cosign
