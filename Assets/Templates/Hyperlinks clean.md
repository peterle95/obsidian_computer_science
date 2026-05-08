<%*
// Clean corrupted notes - remove ALL links and reference numbers while preserving formatting
// Usage: Run this script with Templater in Obsidian

async function cleanCorruptedNotes(tp) {
 const app = tp.app;
 const vault = app.vault;
 // CORRECTED FOLDER PATH:
 const targetFolderPath = "Assets/Put_here_hyperlinknote/"; // Define the target folder

 // Get all markdown files and filter them to include only those in the target folder
 const allFiles = vault.getMarkdownFiles();
 const files = allFiles.filter(file => file.path.startsWith(targetFolderPath));

 let processedCount = 0;
 let cleanedCount = 0;

 if (files.length === 0) {
  new Notice(`No files found in the folder: ${targetFolderPath}`);
  return;
 }

 for (const file of files) {
 try {
 // Read the file content
 const content = await vault.read(file);

 // Skip files with the "to_finish" tag
 if (!content.includes('to_finish')) {
 continue;
 }

 processedCount++;
 console.log(`Processing: ${file.path}`);

 // Check if the file needs cleaning
 if (needsCleaning(content)) {
 const cleanedContent = cleanContent(content);

 // Only update if content actually changed
 if (cleanedContent !== content) {
 await vault.modify(file, cleanedContent);
 cleanedCount++;
 console.log(`Cleaned: ${file.path}`);
 } else {
 console.log(`No changes needed: ${file.path}`);
 }
 }
 } catch (error) {
 console.error(`Error processing ${file.path}:`, error);
 }
 }

 console.log(`\nProcessing complete!`);
 console.log(`Files processed in target folder: ${processedCount}`);
 console.log(`Files cleaned: ${cleanedCount}`);

 // Show a more specific notification in Obsidian
 new Notice(`Cleaned ${cleanedCount} out of ${processedCount} files in '${targetFolderPath}'`);
}

function needsCleaning(content) {
 // Check for any http/https links or orphaned reference numbers
 return /https?:\/\//.test(content) ||
 /\[.*?\]\(.*?\)/.test(content) ||
 /\.\d+\s/.test(content) ||
 /\.\d+$/.test(content) ||
 /\[\d+\]/.test(content);
}

function cleanContent(content) {
 let cleaned = content;

 // Remove ALL markdown links text regardless of URL type
 cleaned = cleaned.replace(/\[([^\]]*)\]\([^)]*\)/g, '$1');

 // Remove ALL standalone URLs (http/https)
 cleaned = cleaned.replace(/https?:\/\/[^\s\)\]\,\.\!]+/g, '');

 // Remove reference-style link definitions : url
 cleaned = cleaned.replace(/^\s*\[\d+\]:\s*.*$/gm, '');

 // Remove reference numbers in square brackets , , etc.
 cleaned = cleaned.replace(/\[\d+\]/g, '');

 // Remove orphaned reference numbers that appear after words
 cleaned = cleaned.replace(/(\w)\.\d+(\s|$)/g, '$1$2');

 // Remove orphaned reference numbers at the end of sentences
 cleaned = cleaned.replace(/\.\d+\./g, '.');

 // Remove standalone reference numbers like at the end
 cleaned = cleaned.replace(/\.\d+$/gm, '');

 // Remove reference numbers in the middle of text
 cleaned = cleaned.replace(/\.\d+\s/g, ' ');

 // Clean up any remaining URL fragments
 cleaned = cleaned.replace(/\(https?:\/\/[^)]*\)/g, '');

 // Remove empty parentheses and brackets
 cleaned = cleaned.replace(/\(\s*\)/g, '');
 cleaned = cleaned.replace(/\[\s*\]/g, '');

 // IMPROVED: Clean up whitespace while preserving structure
 cleaned = cleaned.replace(/[ \t]+/g, ' ');

 // Fix spacing around markdown headers and preserve them
 cleaned = cleaned.replace(/\s*#\s*/g, '\n\n# ');
 cleaned = cleaned.replace(/\s*##\s*/g, '\n\n## ');
 cleaned = cleaned.replace(/\s*###\s*/g, '\n\n### ');

 // Ensure proper paragraph breaks
 cleaned = cleaned.replace(/\.\s+(Core Explanation|Key Characteristics|Purpose\/Why|Examples|Related Concepts)/g, '.\n\n$1');

 // Fix the metadata section at the top
 cleaned = cleaned.replace(/---\s*/g, '\n---\n');

 // Clean up excessive line breaks (more than 2 consecutive)
 cleaned = cleaned.replace(/\n{3,}/g, '\n\n');

 // Remove trailing whitespace from lines
 cleaned = cleaned.replace(/[ \t]+$/gm, '');

 // Ensure there's a line break after the frontmatter
 cleaned = cleaned.replace(/(---[^`]+?---)\s*([A-Z])/g, '$1\n\n$2');

 return cleaned.trim();
}

// Execute the cleaning function
await cleanCorruptedNotes(this);
%>