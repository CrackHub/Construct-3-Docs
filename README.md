# This content is generated for AI Agents.
**Generated Date:** 2026-07-07

**Latest Construct 3 Version:** r492

# Construct Manual Processor

This repository contains a set of scripts designed to download, process, and package the Construct Manual into a portable, offline-friendly format. The goal is to provide a local copy of the manual with standardized file paths and correctly re-written relative internal links, suitable for offline access or deployment as a static website (e.g., using GitHub Pages).

## Project Goals

1.  **Standardize the folder structure**: Remove language-specific subdirectories (e.g., `en/`) and consolidate all content into a single hierarchy.
2.  **Rewrite internal links**: Update all internal links within HTML and Markdown files to be relative paths, ensuring portability when the manual is moved to a local directory (e.g., `C:\Users\X\Downloads`).
3.  **Identify broken links**: Report any links that point to content not present in the locally downloaded manual.
4.  **Create a portable ZIP archive**: Package the processed manual into a ZIP file for easy distribution and local use.

## How It Works

This Colab notebook provides a comprehensive solution for processing the downloaded Construct Manual content. The notebook is divided into two main parts: a Node.js script for initial crawling and basic link processing, and a Python script for further, more robust link normalization and cleanup.

### Part 1: Initial Content Crawling and Basic Processing (Node.js)

This section uses a Node.js script (`crawler.js`) to interact with the Firecrawl API. Its main responsibilities include:

*   **Initiating a crawl**: Fetches content from a specified base URL and domain.
*   **Polling for completion**: Monitors the crawl job's status until all pages are scraped.
*   **Basic URL normalization**: Cleans up URLs by removing language codes (`en/`) and invalid Windows characters.
*   **Initial link updating**: Attempts to adjust internal links to be relative paths based on a simplistic logic.
*   **File saving**: Stores the crawled HTML content into a local directory structure.
*   **ZIP creation**: Compressses the initially processed content into a ZIP archive.

**`crawler.js` Script:**
```javascript
const fs = require('fs');
const path = require('path');

const API_KEY = 'API-KEY'; // Enter your API Key
const BASE_URL = 'https://www.construct.net/en/make-games/manuals';
const BASE_DOMAIN = 'https://www.construct.net';
const BASE_DIR = 'construct_manual';

async function sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

// 1. Start Crawl Job
async function startCrawl() {
    console.log("🚀 Starting crawl via Firecrawl...");
    const response = await fetch('https://api.firecrawl.dev/v2/crawl', {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${API_KEY}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            url: BASE_URL,
            sitemap: "include",
            crawlEntireDomain: false,
            limit: 750, // Limit increased to fetch the entire manual
            scrapeOptions: {
                onlyMainContent: true,
                formats: ["html"],
                proxy: "auto",
                timeout: 60000,
                maxAge: 0,
                excludeTags: [
                    ".manualTopper",
                    ".bottomLinks",
                    ".manualMobileMenuWrap",
                    ".articleItemProps",
                    ".onThisPageWrap",
                    "#LeftMenuWrap",
                    "#ThemeToggler",
                    ".aspNetHidden",
                    "#ReportPopContainer",
                    ".subMenuAdjuster",
                    ".toolbar"
                ]
            }
        })
    });
    const initData = await response.json();
    if (!initData.success) {
        throw new Error(`Crawl failed to start: ${JSON.stringify(initData)}`);
    }
    console.log(`✅ Job ID received: ${initData.id}`);
    return initData.id;
}

// 2. Poll Status Until Complete
async function pollCrawl(jobId) {
    console.log("⏳ Fetching pages, please wait...");
    let allData = [];
    let currentUrl = `https://api.firecrawl.dev/v2/crawl/${jobId}`;

    while (currentUrl) {
        const response = await fetch(currentUrl, {
            headers: { 'Authorization': `Bearer ${API_KEY}` }
        });
        const statusData = await response.json();

        if (statusData.status === 'completed') {
            if (statusData.data) allData = allData.concat(statusData.data);
            currentUrl = statusData.next;
        } else if (statusData.status === 'failed' || statusData.status === 'cancelled') {
            throw new Error(`Crawl failed: ${statusData.status}`);
        } else {
            console.log(`🔄 Status: ${statusData.status} | Progress: ${statusData.completed}/${statusData.total}`);
            await sleep(3000);
        }
    }
    return allData;
}

// 🌟 NEW AND CRITICAL FUNCTION: Normalize URL
function getCleanPath(url) {
    const urlObj = new URL(url);
    let pathStr = urlObj.pathname.replace(/^\/|\/$/g, '');

    // Construct.net sometimes uses '/en/' in links, sometimes not.
    // This regular expression deletes the leading 2-letter language code and slash (e.g., 'en/').
    // This way, both types of links fall into the same folder structure ('make-games/manuals/...').
    pathStr = pathStr.replace(/^[a-z]{2}\//, '');

    if (!pathStr) pathStr = 'index';

    // Clean up invalid characters for Windows
    return pathStr.replace(/[<>":|?*\\]/g, '_');
}

// 3. Convert URL to Windows-Compatible File Path
function urlToFilePath(url) {
    const cleanPath = getCleanPath(url);
    return path.join(BASE_DIR, cleanPath + '.html');
}

// 4. Clean Links within HTML and Convert to Local Paths
function updateLinks(html, currentFilePath) {
    const linkRegex = /<a\s+([^>]*?)href="([^"]+)"([^>]*?)>(.*?)<\/a>/gi;

    return html.replace(linkRegex, (match, beforeHref, url, afterHref, text) => {
        let fullUrl = url;
        let hash = '';

        if (url.includes('#')) {
            const parts = url.split('#');
            fullUrl = parts[0];
            hash = '#' + parts[1];
        }

        try {
            const urlObj = new URL(fullUrl, BASE_DOMAIN);
            if (urlObj.pathname === '/out' && urlObj.searchParams.has('u')) {
                const decodedTarget = decodeURIComponent(urlObj.searchParams.get('u'));
                fullUrl = decodedTarget;
            }
        } catch (e) {}

        if (fullUrl.startsWith(BASE_DOMAIN) || fullUrl.startsWith('/')) {
            if (fullUrl.startsWith('/')) {
                fullUrl = BASE_DOMAIN + fullUrl;
            }

            // 🌟 Also normalize the target file using the same process
            const targetCleanPath = getCleanPath(fullUrl);
            const targetFilePath = path.join(BASE_DIR, targetCleanPath + '.html');

            const currentDir = path.dirname(currentFilePath);
            const relativePath = path.relative(currentDir, targetFilePath);
            const htmlLink = relativePath.replace(/\\/g, '/');

            return `<a ${beforeHref}href="${htmlLink}${hash}"${afterHref}>${text}<\/a>`;
        }

        return `<a ${beforeHref}href="${fullUrl}${hash}"${afterHref}>${text}<\/a>`;
    });
}

// 5. Save Files with Folder Structure
async function saveFiles(pages) {
    console.log(`\n📂 Saving ${pages.length} pages and optimizing links...`);
    for (const page of pages) {
        const url = page.metadata?.sourceURL || page.metadata?.url;
        let html = page.html;

        if (!url || !html) continue;

        const filePath = urlToFilePath(url);
        const dir = path.dirname(filePath);
        fs.mkdirSync(dir, { recursive: true });

        html = updateLinks(html, filePath);
        fs.writeFileSync(filePath, html, 'utf8');
    }
    console.log("✅ All files saved successfully!");
}

// 6. Create ZIP
async function createZip() {
    console.log("📦 Creating ZIP file...");
    try {
        const { execSync } = require('child_process');
        execSync(`zip -r construct_manual.zip ${BASE_DIR}`);
        console.log("✅ ZIP file created: construct_manual.zip");
    } catch (e) {
        console.log("An error occurred while creating the ZIP:", e.message);
    }
}

async function main() {
    try {
        const jobId = await startCrawl();
        const pages = await pollCrawl(jobId);
        await saveFiles(pages);
        await createZip();
    } catch (error) {
        console.error("❌ Error:", error.message);
    }
}

main();
```

### Run the Crawler Script
Execute the Node.js script to start the crawling process. This will download the web content, perform initial processing, and create `construct_manual.zip`.
```bash
!node crawler.js
```

### Download the Crawled Content
This cell initiates the download of the `construct_manual.zip` file, which contains the content initially processed by the Node.js crawler.
```python
from google.colab import files

print("📥 Starting download...")
files.download("construct_manual.zip")
```

---

### Part 2: Advanced Manual Processing and Link Rewriting (Python)

This Python script performs a more rigorous standardization and link-fixing process on the `construct_manual` directory, building upon the initial crawl. It addresses issues like inconsistent directory structures and ensures that all internal links are truly relative and functional when the manual is used offline.

#### Step 1: Standardize Folder Structure

This step iterates through the `construct_manual` directory, identifying files within language-specific subfolders (e.g., `en/`). It moves these files up into the `BASE_DIR`'s main hierarchy and then removes any empty language folders. This ensures a consistent and flat directory structure.

#### Step 2: Map Files and Rewrite Links

This is the core of the script, where all internal links are meticulously processed and rewritten. It involves several sub-steps:

1.  **Create a `file_map`**: Builds a dictionary that maps normalized file paths (without extensions or language codes) to their absolute paths on the disk. This map is crucial for quickly looking up target files.
2.  **Iterate and process links**: Goes through every HTML and Markdown file.
3.  **Resolve URLs**: Uses `urllib.parse.urljoin` to robustly resolve all `href` values (relative, root-relative, absolute) to their full absolute URLs, relative to the current file's URL. This is critical for accurate internal link identification.
4.  **Identify internal vs. external links**: Determines if a resolved URL points to content within the `BASE_DOMAIN`.
5.  **Rewrite internal links**: For internal links, calculates the correct relative path from the current file to the target file using `os.path.relpath` and updates the `href`.
6.  **Handle broken links**: If an internal link target is not found in the `file_map` (meaning it wasn't downloaded), the original full URL is preserved, and the link is recorded as 'broken'.
7.  **Save modified files**: Overwrites the original HTML/Markdown files with the updated links.

#### Step 3: Create Portable ZIP Archive

Finally, the script compresses the fully processed `construct_manual` directory into a new ZIP file named `construct_manual_master.zip`. This archive contains all the standardized files with correctly rewritten internal links, ready for offline use.

**Python Processing Script:**
```python
#!pip install beautifulsoup4 -q

import os
import shutil
import re
import zipfile
from urllib.parse import urlparse, unquote, parse_qs, urljoin
from bs4 import BeautifulSoup
from google.colab import files

BASE_DIR = "construct_manual"
BASE_DOMAIN = "https://www.construct.net"

# ==========================================
# STEP 1: CLEAN UP PHYSICAL FOLDER STRUCTURE
# ==========================================
print("🛠️ STEP 1: Cleaning up language folders like 'en/' and moving files into a single hierarchy...")
for root, dirs, files_list in os.walk(BASE_DIR, topdown=False):
    for file in files_list:
        if file.endswith(('.html', '.md')):
            old_path = os.path.join(root, file)
            rel_path = os.path.relpath(old_path, BASE_DIR).replace('\\', '/')
            parts = rel_path.split('/')

            # If the first folder is a 2-letter language code (e.g., 'en', 'tr')
            if len(parts) > 1 and len(parts[0]) == 2 and parts[0].isalpha():
                new_rel_path = '/'.join(parts[1:])
                new_path = os.path.join(BASE_DIR, new_rel_path)

                os.makedirs(os.path.dirname(new_path), exist_ok=True)
                if os.path.exists(new_path):
                    os.remove(new_path) # Delete old if conflict
                shutil.move(old_path, new_path)

    # Delete remaining empty language folders
    try:
        if not os.listdir(root) and os.path.abspath(root) != os.path.abspath(BASE_DIR):
            os.rmdir(root)
    except OSError:
        pass

print("✅ Physical folder structure standardized.")

# ==========================================
# STEP 2: BUILD NEW MAP AND REWRITE LINKS
# ==========================================
print("🔗 STEP 2: Links are being rewritten from scratch, perfectly compatible with the physical disk...")

file_map = {}
all_files = []

for root, dirs, files_list in os.walk(BASE_DIR):
    for file in files_list:
        if file.endswith(('.html', '.md')):
            abs_path = os.path.abspath(os.path.join(root, file))
            rel_path = os.path.relpath(abs_path, BASE_DIR).replace('\\', '/')

            # Remove extension and normalize
            norm_path = re.sub(r'\.(html|md)$', '', rel_path)
            norm_path = re.sub(r'^[a-z]{2}/', '', norm_path) # Also clean up remaining language codes
            norm_path = norm_path if norm_path else 'index'

            file_map[norm_path] = abs_path
            all_files.append((abs_path, norm_path))

            # index alias
            if norm_path.endswith('/index'):
                file_map[norm_path[:-6]] = abs_path

broken_links = []
fixed_count = 0

def process_file_links(abs_path, current_norm, file_map, BASE_DOMAIN, broken_links):
    global fixed_count

    current_dir = os.path.dirname(abs_path);
    is_html = abs_path.endswith('.html')

    with open(abs_path, 'r', encoding='utf-8') as f:
        content = f.read()

    soup = BeautifulSoup(content, 'html.parser') if is_html else None
    file_modified = False # This variable is in an enclosing function scope for 'replacer'

    # Construct the full URL for the current file being processed
    # This is crucial for correctly resolving relative links using urljoin
    current_file_url = f"{BASE_DOMAIN}/{current_norm}" + ('.html' if is_html else '.md')

    # Find links (<a> for HTML, Regex for MD)
    links = soup.find_all('a', href=True) if is_html else re.finditer(r'\[([^\]]+)\]\(([^)]+)\)', content)

    if is_html:
        for a in links:
            href = a['href']
            hash_part = ''
            if '#' in href:
                href, hash_part = href.split('#', 1)
                hash_part = '#' + hash_part

            if not href: continue

            # Resolve /out?u=...
            try:
                parsed = urlparse(href)
                if parsed.path == '/out' and 'u' in parsed.query:
                    href = unquote(parse_qs(parsed.query)['u'][0])
            except: pass

            # Resolve the href to a full absolute URL relative to the current file's URL
            full_resolved_url = urljoin(current_file_url, href)

            # Now check if this resolved URL belongs to our domain
            is_internal = urlparse(full_resolved_url).netloc == urlparse(BASE_DOMAIN).netloc

            if is_internal:
                target_path = urlparse(full_resolved_url).path.strip('/')
                target_norm = re.sub(r'^[a-z]{2}/', '', target_path)
                target_norm = re.sub(r'\.(html|md)$', '', target_norm) # Crucial fix: Remove extension
                target_norm = target_norm if target_norm else 'index'

                target_abs = file_map.get(target_norm)
                if target_abs:
                    rel = os.path.relpath(target_abs, current_dir).replace('\\', '/')
                    a['href'] = rel + hash_part
                    file_modified = True
                    fixed_count += 1
                else:
                    broken_links.append(target_norm)
                    a['href'] = full_resolved_url + hash_part # Keep original full URL if not found locally
                    file_modified = True

        if file_modified:
            with open(abs_path, 'w', encoding='utf-8') as f:
                f.write(str(soup))

    else: # Markdown
        def replacer(match):
            nonlocal file_modified # Correctly refers to file_modified in process_file_links scope
            global fixed_count     # Correctly refers to the module-level fixed_count

            text, url = match.group(1), match.group(2)
            hash_part = ''
            if '#' in url:
                url, hash_part = url.split('#', 1)
                hash_part = '#' + hash_part

            try:
                parsed = urlparse(url)
                if parsed.path == '/out' and 'u' in parsed.query:
                    url = unquote(parse_qs(parsed.query)['u'][0])
            except: pass

            # Resolve the url to a full absolute URL relative to the current file's URL
            full_resolved_url = urljoin(current_file_url, url)

            # Now check if this resolved URL belongs to our domain
            is_internal = urlparse(full_resolved_url).netloc == urlparse(BASE_DOMAIN).netloc

            if is_internal:
                target_path = urlparse(full_resolved_url).path.strip('/')
                target_norm = re.sub(r'^[a-z]{2}/', '', target_path)
                target_norm = re.sub(r'\.(html|md)$', '', target_norm) # Crucial fix: Remove extension
                target_norm = target_norm if target_norm else 'index'

                target_abs = file_map.get(target_norm)
                if target_abs:
                    rel = os.path.relpath(target_abs, current_dir).replace('\\', '/')
                    file_modified = True
                    fixed_count += 1
                    return f"[{text}]({rel}{hash_part})"
                else:
                    broken_links.append(target_norm)
                    file_modified = True
                    return f"[{text}]({full_resolved_url}{hash_part})"
            return match.group(0)

        new_content = re.sub(r'\[([^\]]+)\]\(([^)]+)\)', replacer, content)
        if file_modified:
            with open(abs_path, 'w', encoding='utf-8') as f:
                f.write(new_content)

for abs_path, current_norm in all_files:
    process_file_links(abs_path, current_norm, file_map, BASE_DOMAIN, broken_links)

print(f"🎉 {fixed_count} links perfectly compatible with the physical disk have been made.")
if broken_links:
    print(f"⚠️ {len(set(broken_links))} broken links (undownloaded pages) were left as original URLs.")

# ==========================================
# STEP 3: CREATE PORTABLE ZIP ARCHIVE
# ==========================================
print("📦 STEP 3: Final, portable ZIP is being created...")
zip_name = "construct_manual_master.zip"
with zipfile.ZipFile(zip_name, 'w', zipfile.ZIP_DEFLATED) as zipf:
    for root, dirs, files_list in os.walk(BASE_DIR):
        for file in files_list:
            file_path = os.path.join(root, file)
            # Ensure only 'make-games/...' structure inside ZIP, preserve 'construct_manual' root
            arcname = os.path.relpath(file_path, os.path.dirname(BASE_DIR))
            zipf.write(file_path, arcname)

print("📥 Starting download...")
files.download(zip_name)
```

### Extract Processed Manual for Verification
This cell extracts the `construct_manual_master.zip` file (generated by the Python script) into a new directory named `construct_manual_fixed`. This step is useful for verifying that all files have been processed correctly and that the relative links are functional in a local environment.
```python
import zipfile
import os

# Assuming the zip file created by the previous script is 'construct_manual_master.zip'
zip_file_to_extract = "construct_manual.zip"
destination_path = "./"

# Create the destination directory if it doesn't exist
os.makedirs(destination_path, exist_ok=True)

# Extract the zip file
with zipfile.ZipFile(zip_file_to_extract, 'r') as zip_ref:
    zip_ref.extractall(destination_path)

print(f"'{zip_file_to_extract}' successfully extracted to '{destination_path}'")
```

---

### Cleanup
Clear Downloaded files
This cell removes the temporary `construct_manual` directory and the `construct_manual_fixed` directory, and optionally the generated `construct_manual.zip` file, to clean up the Colab environment after the processing is complete.
```bash
!rm -rf construct_manual
!rm -rf construct_manual_fixed
#!rm -f construct_manual.zip
```

---

## Output

-   `construct_manual.zip`: Initial archive generated by the Node.js crawler.
-   `construct_manual_master.zip`: Final portable ZIP archive containing the fully processed manual with standardized structure and relative links.

## Publishing to GitHub Pages

Once you have the `construct_manual_master.zip` file, you can easily publish it as a static website using GitHub Pages:

1.  **Unzip the archive**: Extract `construct_manual_master.zip` into a new directory (e.g., `docs/`).
2.  **Create a new GitHub Repository**: Initialize a new public repository on GitHub.
3.  **Upload the `docs/` folder**: Push the contents of your `docs/` folder to the `main` branch of your new repository.
4.  **Configure GitHub Pages**: Go to your repository settings on GitHub, navigate to 'Pages' in the sidebar, and set the source for GitHub Pages to your `main` branch and the `/docs` folder. Your manual will be live at `https://<YOUR_USERNAME>.github.io/<YOUR_REPO_NAME>/`.

Enjoy your offline-friendly Construct Manual!
