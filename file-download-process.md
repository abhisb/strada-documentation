
The download process is **fully automated and happens in real time** during the execution of `import.js`. It does not require any manual download.

Here is exactly how it works programmatically:

### 1. The Automatic Checks and Download Flow
Inside the `processAccidentDataset(year)` method, the script check for files on your disk before taking action:

```javascript
const zipPath = path.join(DATA_DIR, `Unfallorte${year}.zip`);
const csvPath = path.join(DATA_DIR, `Unfallorte${year}.csv`);

if (!fs.existsSync(csvPath)) {
  // If the extracted CSV is not present, check for the ZIP file
  if (!fs.existsSync(zipPath)) {
    // If the ZIP is also missing, fetch it from the official servers in real time
    const url = `https://www.opengeodata.nrw.de/produkte/transport_verkehr/unfallatlas/Unfallorte${year}_EPSG25832_CSV.zip`;
    await downloadFile(url, zipPath);
  }
  
  // Extract ZIP and save the CSV to data/ folder...
}
```

### 2. Sourcing & Extraction Step-by-Step
1. **Request Sourcing**: If the CSV is not found, the script automatically triggers an HTTPS `GET` request using the `downloadFile(url, zipPath)` helper to fetch the corresponding ZIP from the German open-data portal (`opengeodata.nrw.de`).
2. **ZIP Extraction**: Once the download completes, the script utilizes the `AdmZip` library to locate the `.csv` or `.txt` record layout inside the zip file.
3. **Local Caching (Bypassing Abuse)**: If you run the script again, the check `fs.existsSync(csvPath)` returns `true`, and it **skips the network download entirely**. This ensures we do not abuse the external geodata server by repeatedly downloading the heavy ZIP files (each zip is around 10MB to 15MB).