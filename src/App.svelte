<script lang="ts">
  import JSZip from "jszip";

  let files: FileList | null = null;
  let sanitizedNames: { original: string; sanitized: string }[] = [];
  let isProcessing = false;
  let showResults = false;
  let showNotification = false;
  let isLoading = false;
  let notificationMessage = "";

  // Options
  let sanitizeNames = true;
  let convertToAscii = false;
  let convertToBinary = false;

  function sanitizeFileName(fileName: string): string {
    const nameWithoutExt = fileName.replace(/\.stl$/i, "");
    const sanitized = nameWithoutExt.replace(/[^a-zA-Z0-9_]/g, "_");
    return sanitized + ".stl";
  }

  function handleDuplicates(names: string[]): string[] {
    const nameCount = new Map<string, number>();
    const result: string[] = [];

    for (const name of names) {
      if (!nameCount.has(name)) {
        nameCount.set(name, 0);
        result.push(name);
      } else {
        const count = nameCount.get(name)! + 1;
        nameCount.set(name, count);
        const newName = name.replace(".stl", `_${count}.stl`);
        result.push(newName);
      }
    }

    return result;
  }

  function handleFileSelect(event: Event) {
    const target = event.target as HTMLInputElement;
    files = target.files;
    showResults = false;
  }

  async function processFiles() {
    if (!files || files.length === 0) return;

    isLoading = true;
    showNotification = true;
    notificationMessage = "Processing files...";
    isProcessing = true;
    showResults = false;

    // Prepare sanitized names if option is selected
    if (sanitizeNames) {
      const sanitized = Array.from(files).map((file) =>
        sanitizeFileName(file.name),
      );
      const uniqueNames = handleDuplicates(sanitized);

      sanitizedNames = Array.from(files).map((file, index) => ({
        original: file.name,
        sanitized: uniqueNames[index],
      }));
    } else {
      sanitizedNames = Array.from(files).map((file) => ({
        original: file.name,
        sanitized: file.name,
      }));
    }

    // Small delay to show notification
    await new Promise((resolve) => setTimeout(resolve, 500));

    showNotification = false;
    showResults = true;
    isProcessing = false;
    isLoading = false;
  }

  async function downloadAsZip() {
    if (!files || files.length === 0) return;

    isLoading = true;
    showNotification = true;
    notificationMessage = "Creating ZIP file...";
    isProcessing = true;
    const zip = new JSZip();

    for (let i = 0; i < files.length; i++) {
      const file = files[i];
      const fileName = sanitizedNames[i].sanitized;
      let convertedData: string | ArrayBuffer = await file.arrayBuffer();

      // Apply conversion options
      if (convertToAscii) {
        convertedData = await convertSTLToAscii(file);
      } else if (convertToBinary) {
        convertedData = await convertSTLToBinary(file);
      }

      zip.file(fileName, convertedData);
    }

    const blob = await zip.generateAsync({ type: "blob" });
    const url = URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.href = url;
    link.download = "sanitized_stl_files.zip";
    link.click();
    URL.revokeObjectURL(url);

    showNotification = false;
    isProcessing = false;
    isLoading = false;
  }

  async function convertSTLToAscii(file: File): Promise<string | ArrayBuffer> {
    const fileData = await file.arrayBuffer();
    const dataView = new DataView(fileData);

    const textDecoder = new TextDecoder();
    const first5Bytes = new Uint8Array(
      dataView.buffer,
      0,
      Math.min(5, dataView.byteLength),
    );
    const header = textDecoder.decode(first5Bytes);

    if (header.toLowerCase().startsWith("solid")) {
      return fileData;
    }

    let ascii = "solid converted\n";
    let offset = 80;

    const numTriangles = dataView.getUint32(offset, true);
    offset += 4;

    for (let i = 0; i < numTriangles; i++) {
      const nx = dataView.getFloat32(offset, true);
      offset += 4;
      const ny = dataView.getFloat32(offset, true);
      offset += 4;
      const nz = dataView.getFloat32(offset, true);
      offset += 4;

      ascii += `  facet normal ${nx} ${ny} ${nz}\n`;
      ascii += "    outer loop\n";

      for (let v = 0; v < 3; v++) {
        const vx = dataView.getFloat32(offset, true);
        offset += 4;
        const vy = dataView.getFloat32(offset, true);
        offset += 4;
        const vz = dataView.getFloat32(offset, true);
        offset += 4;
        ascii += `      vertex ${vx} ${vy} ${vz}\n`;
      }

      ascii += "    endloop\n";
      ascii += "  endfacet\n";
      offset += 2;
    }

    ascii += "endsolid converted\n";
    return ascii;
  }

  async function convertSTLToBinary(file: File): Promise<ArrayBuffer> {
    const fileData = await file.arrayBuffer();
    const dataView = new DataView(fileData);

    const textDecoder = new TextDecoder();
    const first5Bytes = new Uint8Array(
      dataView.buffer,
      0,
      Math.min(5, dataView.byteLength),
    );
    const header = textDecoder.decode(first5Bytes);

    // If already binary, return as is
    if (!header.toLowerCase().startsWith("solid")) {
      return fileData;
    }

    // Parse ASCII STL
    const text = textDecoder.decode(new Uint8Array(fileData));
    const facetRegex =
      /facet normal\s+([-\d.e+]+)\s+([-\d.e+]+)\s+([-\d.e+]+)[\s\S]*?vertex\s+([-\d.e+]+)\s+([-\d.e+]+)\s+([-\d.e+]+)[\s\S]*?vertex\s+([-\d.e+]+)\s+([-\d.e+]+)\s+([-\d.e+]+)[\s\S]*?vertex\s+([-\d.e+]+)\s+([-\d.e+]+)\s+([-\d.e+]+)/g;

    const triangles = [];
    let match;
    while ((match = facetRegex.exec(text)) !== null) {
      triangles.push({
        normal: [
          parseFloat(match[1]),
          parseFloat(match[2]),
          parseFloat(match[3]),
        ],
        vertices: [
          [parseFloat(match[4]), parseFloat(match[5]), parseFloat(match[6])],
          [parseFloat(match[7]), parseFloat(match[8]), parseFloat(match[9])],
          [parseFloat(match[10]), parseFloat(match[11]), parseFloat(match[12])],
        ],
      });
    }

    // Create binary STL
    const numTriangles = triangles.length;
    const bufferSize = 84 + numTriangles * 50; // Header + triangles
    const buffer = new ArrayBuffer(bufferSize);
    const view = new DataView(buffer);

    // Write header (80 bytes)
    const headerBytes = new TextEncoder().encode(
      "Binary STL converted from ASCII",
    );
    for (let i = 0; i < Math.min(80, headerBytes.length); i++) {
      view.setUint8(i, headerBytes[i]);
    }

    // Write number of triangles
    view.setUint32(80, numTriangles, true);

    let offset = 84;
    for (const triangle of triangles) {
      // Normal
      view.setFloat32(offset, triangle.normal[0], true);
      offset += 4;
      view.setFloat32(offset, triangle.normal[1], true);
      offset += 4;
      view.setFloat32(offset, triangle.normal[2], true);
      offset += 4;

      // Vertices
      for (const vertex of triangle.vertices) {
        view.setFloat32(offset, vertex[0], true);
        offset += 4;
        view.setFloat32(offset, vertex[1], true);
        offset += 4;
        view.setFloat32(offset, vertex[2], true);
        offset += 4;
      }

      // Attribute byte count (0)
      view.setUint16(offset, 0, true);
      offset += 2;
    }

    return buffer;
  }
</script>

<main>
  <h1>STL File Name Sanitizer</h1>

  <p class="description">
    Clean up your STL file names and convert between ASCII and binary formats.
    Remove spaces and special characters, ensuring compatibility across
    different systems.
  </p>

  <label class="file-button">
    <input type="file" accept=".stl" multiple on:change={handleFileSelect} />
    Choose STL Files
  </label>

  {#if files && files.length > 0}
    <div class="options-row">
      <div class="options">
        <label class="checkbox-label">
          <input type="checkbox" bind:checked={sanitizeNames} />
          <span>Sanitize Names</span>
        </label>

        <label class="checkbox-label">
          <input
            type="checkbox"
            bind:checked={convertToAscii}
            on:change={() => {
              if (convertToAscii) convertToBinary = false;
            }}
          />
          <span>Convert to ASCII</span>
        </label>

        <label class="checkbox-label">
          <input
            type="checkbox"
            bind:checked={convertToBinary}
            on:change={() => {
              if (convertToBinary) convertToAscii = false;
            }}
          />
          <span>Convert to Binary</span>
        </label>
      </div>

      <button
        class="convert-button"
        on:click={processFiles}
        disabled={isProcessing}
      >
        Convert Now
      </button>
    </div>
  {/if}

  {#if isLoading}
    <div class="overlay">
      <div class="notification-modern">
        <div class="spinner"></div>
        <div class="notification-text">{notificationMessage}</div>
      </div>
    </div>
  {/if}

  {#if showNotification}
    <div class="notification">
      {notificationMessage}
    </div>
  {/if}

  {#if showResults && sanitizeNames}
    <div class="results">
      <h2>Sanitized File Names:</h2>
      <table>
        <thead>
          <tr>
            <th>Original Name</th>
            <th>Sanitized Name</th>
          </tr>
        </thead>
        <tbody>
          {#each sanitizedNames as item}
            <tr>
              <td>{item.original}</td>
              <td class="sanitized">{item.sanitized}</td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>
  {/if}

  {#if showResults}
    <button
      class="download-button"
      on:click={downloadAsZip}
      disabled={isProcessing}
    >
      {isProcessing ? "Creating ZIP..." : "Download as ZIP"}
    </button>
  {/if}
</main>

<style>
  :global(body) {
    background-color: #000;
    color: #fff;
    font-family: "Cascadia Mono", "Cascadia Code", monospace;
    margin: 0;
    padding: 0;
  }

  main {
    padding: 1.5rem 2rem 2rem;
    max-width: 800px;
    margin: 0 auto;
  }

  h1 {
    margin-bottom: 0.5rem;
    color: #00adad;
  }

  .description {
    margin-bottom: 2rem;
    color: #aaa;
    font-size: 0.9rem;
    line-height: 1.5;
  }

  h2 {
    color: #00adad;
    font-size: 1.2rem;
  }

  .options-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 2rem;
    margin-top: 1.5rem;
    flex-wrap: wrap;
  }

  .options {
    display: flex;
    gap: 2rem;
    flex-wrap: wrap;
  }

  .checkbox-label {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    cursor: pointer;
    font-size: 0.95rem;
    position: relative;
  }

  .checkbox-label input[type="checkbox"] {
    appearance: none;
    -webkit-appearance: none;
    width: 22px;
    height: 22px;
    border: 2px solid #00adad;
    border-radius: 6px;
    cursor: pointer;
    position: relative;
    transition: all 0.3s ease;
    background-color: transparent;
  }

  .checkbox-label input[type="checkbox"]:hover {
    border-color: #00c9c9;
    background-color: rgba(0, 173, 173, 0.1);
  }

  .checkbox-label input[type="checkbox"]:checked {
    background-color: #00adad;
    border-color: #00adad;
  }

  .checkbox-label input[type="checkbox"]:checked::after {
    content: "";
    position: absolute;
    left: 6px;
    top: 2px;
    width: 5px;
    height: 10px;
    border: solid #000;
    border-width: 0 2px 2px 0;
    transform: rotate(45deg);
  }

  .checkbox-label span {
    user-select: none;
  }

  .convert-button {
    padding: 0.75rem 1.5rem;
    background-color: #00adad;
    color: #000;
    border: none;
    border-radius: 4px;
    font-family: "Cascadia Mono", "Cascadia Code", monospace;
    font-size: 1rem;
    font-weight: bold;
    cursor: pointer;
    transition: background-color 0.2s;
    white-space: nowrap;
  }

  .convert-button:hover:not(:disabled) {
    background-color: #00c9c9;
  }

  .convert-button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  .overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.7);
    backdrop-filter: blur(10px);
    -webkit-backdrop-filter: blur(10px);
    z-index: 999;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .notification-modern {
    background: rgba(0, 173, 173, 0.15);
    backdrop-filter: blur(20px);
    -webkit-backdrop-filter: blur(20px);
    border: 2px solid rgba(0, 173, 173, 0.5);
    padding: 2rem 3rem;
    border-radius: 16px;
    box-shadow: 0 8px 32px rgba(0, 173, 173, 0.3);
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1.5rem;
  }

  .notification-text {
    color: #fff;
    font-weight: bold;
    font-size: 1.2rem;
  }

  .spinner {
    width: 50px;
    height: 50px;
    border: 4px solid rgba(255, 255, 255, 0.2);
    border-top: 4px solid #fff;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }

  @keyframes spin {
    0% {
      transform: rotate(0deg);
    }
    100% {
      transform: rotate(360deg);
    }
  }

  .results {
    margin-top: 2rem;
  }

  table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 1rem;
    background-color: #000;
  }

  th,
  td {
    padding: 0.75rem;
    text-align: left;
    border: 1px solid #fff;
  }

  th {
    background-color: #00adad;
    color: #000;
    font-weight: bold;
  }

  td {
    color: #fff;
  }

  td.sanitized {
    color: #00adad;
  }

  .download-button {
    margin-top: 1.5rem;
    padding: 0.75rem 1.5rem;
    background-color: #00adad;
    color: #000;
    border: none;
    border-radius: 4px;
    font-family: "Cascadia Mono", "Cascadia Code", monospace;
    font-size: 1rem;
    font-weight: bold;
    cursor: pointer;
    transition: background-color 0.2s;
  }

  .download-button:hover:not(:disabled) {
    background-color: #00c9c9;
  }

  .download-button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  .file-button {
    display: inline-block;
    padding: 0.75rem 1.5rem;
    background-color: #00adad;
    color: #000;
    border-radius: 4px;
    font-family: "Cascadia Mono", "Cascadia Code", monospace;
    font-size: 1rem;
    font-weight: bold;
    cursor: pointer;
    transition: background-color 0.2s;
  }

  .file-button:hover {
    background-color: #00c9c9;
  }

  .file-button input[type="file"] {
    display: none;
  }
</style>
