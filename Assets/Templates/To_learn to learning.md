<%*
// Script to find notes with today's review date, remove "to_learn" tag, add "learning" tag, and update review date
async function updateNotesForTodaysReview() {
    const files = this.app.vault.getMarkdownFiles();
    let processedCount = 0;
    
	    // Use this code if one day late
    /*let today = new Date();
	today.setDate(today.getDate - 1);
    const todayString = today.getFullYear() + '-' + 
                       String(today.getMonth() + 1).padStart(2, '0') + '-' + 
                       String(today.getDate()).padStart(2, '0');*/
                       
	// Get today's date in YYYY-MM-DD format (fixed timezone issue)
	let today = new Date();
    const todayString = today.getFullYear() + '-' + 
        String(today.getMonth() + 1).padStart(2, '0') + '-' + 
    String(today.getDate()).padStart(2, '0');
    // Function to calculate next review date (10+ days in future, multiple of 5)
    function getNextReviewDate(currentDateString) {
        const date = new Date(currentDateString + 'T00:00:00');
        // Add 10 days
        date.setDate(date.getDate() + 10);
        
        // Get the day of the month
        let day = date.getDate();
        
        // Round up to next multiple of 5
        const nextMultipleOf5 = Math.ceil(day / 5) * 5;
        
        // Handle month overflow
        const lastDayOfMonth = new Date(date.getFullYear(), date.getMonth() + 1, 0).getDate();
        
        if (nextMultipleOf5 > lastDayOfMonth) {
            // Move to next month, day 5
            date.setMonth(date.getMonth() + 1);
            date.setDate(5);
        } else {
            date.setDate(nextMultipleOf5);
        }
        
        return date.getFullYear() + '-' + 
               String(date.getMonth() + 1).padStart(2, '0') + '-' + 
               String(date.getDate()).padStart(2, '0');
    }
    
    // Filter files to only include those in "computer science" folder
    const csFiles = files.filter(file => 
        file.path.toLowerCase().startsWith('computer science/')
    );
    
    if (csFiles.length === 0) {
        new Notice('No files found in "computer science" folder');
        return;
    }
    
    console.log(`Found ${csFiles.length} files in computer science folder`);
    console.log(`Looking for review-date: ${todayString}`);
    
    for (const file of csFiles) {
        try {
            const content = await this.app.vault.read(file);
            
            // Check if file has frontmatter and contains today's review date
            if (content.startsWith('---')) {
                const frontmatterEnd = content.indexOf('---', 3);
                if (frontmatterEnd !== -1) {
                    const frontmatter = content.substring(0, frontmatterEnd + 3);
                    const restOfContent = content.substring(frontmatterEnd + 3);
                    
                    // Check if frontmatter contains today's review date
                    if (frontmatter.includes(`review-date: ${todayString}`)) {
                        console.log(`Found file with today's review date: ${file.path}`);
                        
                        // Calculate next review date
                        const nextReviewDate = getNextReviewDate(todayString);
                        console.log(`Next review date will be: ${nextReviewDate}`);
                        
                        // Replace "to_learn" with "learning" in the frontmatter
                        let updatedFrontmatter = frontmatter
                            .replace(/- to_learn/g, '- learning')  // Array format
                            .replace(/tags: to_learn/g, 'tags: learning')  // Single tag format
                            .replace(/tags: \[to_learn\]/g, 'tags: [learning]')  // Bracket format
                            .replace(/tags: \[([^\]]*),?\s*to_learn\s*,?([^\]]*)\]/g, (match, before, after) => {
                                // Handle to_learn in middle of array
                                let newTags = [before, after].filter(Boolean).join(', ');
                                if (newTags) {
                                    return `tags: [${newTags}, learning]`;
                                } else {
                                    return 'tags: [learning]';
                                }
                            })
                            // Update the review date
                            .replace(`review-date: ${todayString}`, `review-date: ${nextReviewDate}`);
                        
                        const updatedContent = updatedFrontmatter + restOfContent;
                        
                        // Only update if content actually changed
                        if (updatedContent !== content) {
                            await this.app.vault.modify(file, updatedContent);
                            processedCount++;
                            console.log(`Updated: ${file.path} - New review date: ${nextReviewDate}`);
                        }
                    }
                }
            }
        } catch (error) {
            console.error(`Error processing ${file.path}:`, error);
        }
    }
    
    new Notice(`Updated ${processedCount} files scheduled for review today. Changed "to_learn" to "learning" and updated review dates.`);
    console.log(`Review update complete. ${processedCount} files updated for today's date: ${todayString}`);
}

// Run the function
updateNotesForTodaysReview();
%>