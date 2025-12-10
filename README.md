#!/usr/bin/env node

/**
 * pack-folder.js
 *
 * Usage:
 *   node pack-folder.js ./my-folder
 *   node pack-folder.js ./my-folder output.zip meta.json
 */

import fs from "fs";
import crypto from "crypto";
import archiver from "archiver";
import path from "path";

async function zipFolder(source, outPath) {
  const archive = archiver("zip", { zlib: { level: 9 } });
  const stream = fs.createWriteStream(outPath);

  return new Promise((resolve, reject) => {
    archive
      .directory(source, false)
      .on("error", (err) => reject(err))
      .pipe(stream);

    stream.on("close", () => resolve());
    archive.finalize();
  });
}

function sha256OfFile(filePath) {
  const hash = crypto.createHash("sha256");
  const data = fs.readFileSync(filePath);
  hash.update(data);
  return hash.digest("hex");
}

async function main() {
  const folder = process.argv[2];
  if (!folder) {
    console.log("Usage: node pack-folder.js <folder> [zipName] [metaName]");
    process.exit(1);
  }

  const zipName = process.argv[3] || "package.zip";
  const metaName = process.argv[4] || "metadata.json";

  console.log(`Zipping folder: ${folder}`);
  await zipFolder(folder, zipName);

  const sha = sha256OfFile(zipName);
  const stats = fs.statSync(zipName);

  const meta = {
    folder: path.resolve(folder),
    zip: path.resolve(zipName),
    size_bytes: stats.size,
    size_kb: (stats.size / 1024).toFixed(2),
    sha256: sha,
    created_at: new Date().toISOString()
  };

  fs.writeFileSync(metaName, JSON.stringify(meta, null, 2));

  console.log("Done!");
  console.log("ZIP:", zipName);
  console.log("SHA-256:", sha);
}

main();

