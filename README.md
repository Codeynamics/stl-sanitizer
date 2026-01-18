# STL File Sanitizer

> **Live Demo**: [https://codeynamics.github.io/stl-sanitizer/](https://codeynamics.github.io/stl-sanitizer/)

A modern web-based tool to clean up STL file names and convert between ASCII and binary formats. Built with Svelte and TypeScript.

## Features

- **Name Sanitization**: Remove spaces and special characters from STL filenames
- **Format Conversion**: Convert between ASCII and binary STL formats
- **Smart Naming**: Automatically fix `solid` and `endsolid` names in ASCII STL files
- **Batch Processing**: Handle multiple files at once
- **ZIP Export**: Download all processed files as a single ZIP archive
- **Modern UI**: Clean, dark-themed interface with glassmorphism effects

## Usage

> Visit [https://codeynamics.github.io/stl-sanitizer/](https://codeynamics.github.io/stl-sanitizer/) 

Or run locally:
1. **Choose STL Files**: Click "Choose STL Files" to select one or more STL files
2. **Select Options**:
   - **Sanitize Names**: Remove spaces and special characters (replaces with underscores)
   - **Convert to ASCII**: Convert binary STL files to ASCII format
   - **Convert to Binary**: Convert ASCII STL files to binary format
   - **Fix Solid Names**: Update `solid` and `endsolid` lines to match filename
3. **Convert**: Click "Convert Now" to process your files
4. **Download**: Review the changes and download your processed files as a ZIP

## => Options Explained

### Sanitize Names
Cleans up filenames by:
- Removing spaces and special characters
- Keeping only alphanumeric characters and underscores
- Handling duplicate names by adding `_1`, `_2`, etc.

### Convert to ASCII
Converts binary STL files to human-readable ASCII format. Automatically applies solid name fixing.

### Convert to Binary
Converts ASCII STL files to compact binary format for smaller file sizes.

### Fix Solid Names
Updates the `solid` and `endsolid` declarations in ASCII STL files to match the filename. Handles multiple solids in a single file by appending `_1`, `_2`, etc.

## Development

### Prerequisites
- Node.js v18 or higher

### Setup
```bash
# Clone the repository
git clone https://github.com/Codeynamics/stl-sanitizer.git
cd stl-sanitizer

# Install dependencies
npm install

# Run development server
npm run dev
```

### Build for Production
```bash
npm run build
```

### Deploy to GitHub Pages
This project is configured for automatic deployment to GitHub Pages via GitHub Actions. Simply push to the `main` branch and the site will be automatically built and deployed.

## Technologies Used

- **Svelte**: Reactive UI framework
- **TypeScript**: Type-safe JavaScript
- **Vite**: Fast build tool
- **JSZip**: ZIP file generation

## Browser Compatibility

Works on all modern browsers that support:
- ES6+ JavaScript
- File API
- ArrayBuffer
- Blob URLs

## License

MIT License - feel free to use this project for any purpose.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Acknowledgments

Built with ❤️ for the 3D printing and simulation community
