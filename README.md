# spotify-canvas

show your recently played tracks on spotify with a canvas.

## requirements

* NodeJS 16.x
* Yarn
* Vercel CLI
* Spotify client id, client secret, and refresh token

## install

```shell
yarn install 
```

## local development

```shell
git clone https://github.com/umarhadi/spotify-canvas.git
```

```shell
cd spotify-canvas
```

```shell
cp .env.example .env
```

change the values in the .env file with yours.

```shell
yarn build && vercel dev
```

## deploy to Vercel

```shell
yarn build && vercel --prod
```

set the environment variables in your project settings.


