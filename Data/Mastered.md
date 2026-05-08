# ⏱️Random
```dataviewjs
// Get current date as seed for consistent daily randomization
const today = new Date().toISOString().split('T')[0];
// Fetch all notes from Computer Science folder with mastered tag
// Note: Using the correct folder name and tag format
const notes = dv.pages('"Computer Science"')
    .where(p => p.tags && p.tags.includes("mastered"));
// Simple hash function for reproducible randomization
function simpleHash(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
        const char = str.charCodeAt(i);
        hash = ((hash << 5) - hash) + char;
        hash = hash & hash; // Convert to 32-bit integer
    }
    return Math.abs(hash);
}
// Create seeded random selection
function getRandomNotes(notes, seed, count) {
    const notesArray = Array.from(notes);
    if (notesArray.length === 0) return [];
    
    const seededRandom = simpleHash(seed + notesArray.length);
    
    // Shuffle array based on seed
    for (let i = notesArray.length - 1; i > 0; i--) {
        const j = (seededRandom + i) % (i + 1);
        [notesArray[i], notesArray[j]] = [notesArray[j], notesArray[i]];
    }
    
    return notesArray.slice(0, count);
}
// Get 5 random notes for today
const randomNotes = getRandomNotes(notes, today, 5);
// Display results
if (randomNotes.length > 0) {
    dv.header(3, `📚 Today's Random Mastered Notes (${today})`);
    dv.table(
        ["File", "Language", "Last Reviewed"],
        randomNotes.map(note => [
            dv.fileLink(note.file.path, false, note.file.name),
            note.language ? (Array.isArray(note.language) ? note.language.join(", ") : note.language) : "",
            note["last-reviewed"] ? dv.date(note["last-reviewed"]) : ""
        ])
    );
} else {
    dv.paragraph("No notes found with the 'mastered' tag in the Computer Science folder.");
    
    // Debug information
    const allNotes = dv.pages('"Computer Science"');
    dv.paragraph(`Total notes in Computer Science folder: ${allNotes.length}`);
    
    if (allNotes.length > 0) {
        dv.paragraph("Sample of available tags:");
        const sampleNote = allNotes.first();
        dv.paragraph(`Sample note tags: ${sampleNote.tags ? JSON.stringify(sampleNote.tags) : "No tags found"}`);
    }
}
```

# 🧠 Mastered
```dataview
TABLE last-reviewed, language
FROM "Computer Science"
WHERE contains(tags, "mastered") AND last-reviewed 
SORT last-reviewed ASC
```
