<script lang="ts">
  import JSZip from "jszip";

  let files: FileList | null = null;
  let sanitizedNames: { original: string; sanitized: string }[] = [];
  let isProcessing = false;

  function sanitizeFileName(fileName: string): string {
    const nameWithoutExt = fileName.replace(/\.stl$/i, "");
    const sanitized = nameWithoutExt.replace(/[^a-zA-Z0-9_]/g, "_");
    return sanitized + ".stl";
  }

  function handleFileSelect(event: Event) {
    const target = event.target as HTMLInputElement;
    files = target.files;

    if (files) {
      sanitizedNames = Array.from(files).map((file) => ({
        original: file.name,
        sanitized: sanitizeFileName(file.name),
      }));
    }
  }

  async function downloadAsZip() {
    if (!files || files.length === 0) return;

    isProcessing = true;
    const zip = new JSZip();

    for (let i = 0; i < files.length; i++) {
      const file = files[i];
      const sanitized = sanitizeFileName(file.name);
      const convertedData = await convertSTLToAscii(file);
      zip.file(sanitized, convertedData);
    }

    const blob = await zip.generateAsync({ type: "blob" });
    const url = URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.href = url;
    link.download = "sanitized_stl_files.zip";
    link.click();
    URL.revokeObjectURL(url);

    isProcessing = false;
  }

  async function convertSTLToAscii(file: File): Promise<string | ArrayBuffer> {
    const fileData = await file.arrayBuffer();
    const dataView = new DataView(fileData);

    // Check if file is binary STL
    const textDecoder = new TextDecoder();
    const first5Bytes = new Uint8Array(
      dataView.buffer,
      0,
      Math.min(5, dataView.byteLength),
    );
    const header = textDecoder.decode(first5Bytes);

    // If it starts with "solid", it's already ASCII
    if (header.toLowerCase().startsWith("solid")) {
      return fileData;
    }

    // Convert binary to ASCII
    let ascii = "solid converted\n";
    let offset = 80; // Skip 80-byte header

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
      offset += 2; // Skip attribute byte count
    }

    ascii += "endsolid converted\n";
    return ascii;
  }
</script>

<main>
  <h1>STL File Name Sanitizer</h1>

  <label class="file-button">
    <input type="file" accept=".stl" multiple on:change={handleFileSelect} />
    Choose STL Files
  </label>

  {#if sanitizedNames.length > 0}
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

      <button on:click={downloadAsZip} disabled={isProcessing}>
        {isProcessing ? "Creating ZIP..." : "Download as ZIP"}
      </button>
    </div>
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
    padding: 2rem;
    max-width: 800px;
    margin: 0 auto;
  }

  h1 {
    margin-bottom: 2rem;
    color: #00adad;
  }

  h2 {
    color: #00adad;
    font-size: 1.2rem;
  }

  input[type="file"] {
    background-color: #fff;
    color: #000;
    padding: 0.5rem;
    border: 2px solid #00adad;
    border-radius: 4px;
    cursor: pointer;
    font-family: "Cascadia Mono", "Cascadia Code", monospace;
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

  button {
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

  button:hover:not(:disabled) {
    background-color: #00c9c9;
  }

  button:disabled {
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
