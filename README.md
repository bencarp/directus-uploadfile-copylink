# Upload & Copy Links

A Directus module for uploading one or more files and immediately copying their public download URLs.

## Features

- File picker and drag-and-drop uploads
- Parallel multi-file uploads
- Individual and bulk link copying
- Per-file upload and error states
- Responsive layout with Directus theme support

## Installation

Build the extension with `npm run build`, then place this package in your Directus extensions directory and restart Directus.

## Download URLs

Links use the Project URL configured in Directus Project Settings and follow the format `{Project URL}/downloads/{id}`. A Project URL must be configured before files can be uploaded through the module. The project server must be configured to rewrite requests at example.com/downloads/{id} to directus.example.com/assets/{id}.

## Development

- `npm run dev` builds in watch mode.
- `npm run build` creates the production bundle.
- `npm run validate` validates the extension package.

## License

MIT
