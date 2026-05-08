<%*
// Script to replace "learning" tag with "to_learn" tag in YAML frontmatter within "computer science" folder
async function replaceLearningTagInCSFolder() {
    const files = this.app.vault.getMarkdownFiles();
    let processedCount = 0;
    // Filter files to only include those in "computer science" folder
    const csFiles = files.filter(file => 
        file.path.toLowerCase().startsWith('computer science/')
    );
    
    if (csFiles.length === 0) {
        new Notice('No files found in "computer science" folder');
        return;
    }
    
    console.log(`Found ${csFiles.length} files in computer science folder`);
    
    for (const file of csFiles) {
        try {
            const content = await this.app.vault.read(file);
            
            // Check if file has frontmatter and contains "learning" tag
            if (content.startsWith('---')) {
                const frontmatterEnd = content.indexOf('---', 3);
                if (frontmatterEnd !== -1) {
                    const frontmatter = content.substring(0, frontmatterEnd + 3);
                    const restOfContent = content.substring(frontmatterEnd + 3);
                    
                    // Check if frontmatter contains "learning" tag
                    if (frontmatter.includes('learning')) {
                        // Replace "learning" with "to_learn" in the frontmatter
                        // Handle both array format and single line format
                        let updatedFrontmatter = frontmatter
                            .replace(/- learning/g, '- to_learn')  // Array format
                            .replace(/tags: learning/g, 'tags: to_learn')  // Single tag format
                            .replace(/tags: \[learning\]/g, 'tags: [to_learn]')  // Bracket format
                            .replace(/tags: \[([^\]]*),?\s*learning\s*,?([^\]]*)\]/g, (match, before, after) => {
                                // Handle learning in middle of array
                                let newTags = [before, after].filter(Boolean).join(', ');
                                if (newTags) {
                                    return `tags: [${newTags}, to_learn]`;
                                } else {
                                    return 'tags: [to_learn]';
                                }
                            });
                        
                        const updatedContent = updatedFrontmatter + restOfContent;
                        
                        // Only update if content actually changed
                        if (updatedContent !== content) {
                            await this.app.vault.modify(file, updatedContent);
                            processedCount++;
                            console.log(`Updated: ${file.path}`);
                        }
                    }
                }
            }
        } catch (error) {
            console.error(`Error processing ${file.path}:`, error);
        }
    }
    
    new Notice(`Processed ${processedCount} files in computer science folder. Replaced "learning" with "to_learn".`);
    console.log(`Tag replacement complete. ${processedCount} files updated.`);
}

// Run the function
replaceLearningTagInCSFolder();
%>