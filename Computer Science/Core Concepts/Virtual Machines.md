---
memory: to_finish
tags:
  - learned
language:
  - Core Concepts
review-date:
last-reviewed: 2025-09-13
scheda: done
visit-count: 2
confidence-level: 2
consecutive-correct: 2
---

```dataviewjs
const currentPage = dv.current();
let visitCount = currentPage.file.frontmatter["visit-count"] || 0;
let confidence = currentPage.file.frontmatter["confidence-level"] || 1;
let streak = currentPage.file.frontmatter["consecutive-correct"] || 0;

const container = this.container.createEl('div');
container.style.cssText = `
    background: #2a2a2a; border: 1px solid #404040; border-radius: 6px;
    padding: 12px; margin: 10px 0; display: inline-block;
`;

// Status display
const status = container.createEl('div');
status.innerHTML = `
    <strong>Learning Progress</strong><br>
    Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
`;
status.style.cssText = 'margin-bottom: 10px; font-size: 13px; color: #cccccc;';

// Quick feedback buttons
const buttonContainer = container.createEl('div');
['Got it! ✅', 'Struggled ⚠️', 'Failed ❌'].forEach((label, index) => {
    const btn = buttonContainer.createEl('button');
    btn.textContent = label;
    btn.style.cssText = `
        margin-right: 8px; padding: 4px 8px; border: none; border-radius: 3px;
        cursor: pointer; font-size: 11px;
        background: ${['#28a745', '#ffc107', '#dc3545'][index]};
        color: ${index === 1 ? '#000' : '#fff'};
    `;
    
    btn.addEventListener('click', async () => {
        visitCount++;
        if (index === 0) { // Got it
            confidence = Math.min(5, confidence + 0.5);
            streak++;
        } else { // Struggled or failed
            confidence = Math.max(1, confidence - 0.5);
            streak = 0;
        }
        
        const file = app.vault.getAbstractFileByPath(currentPage.file.path);
        await app.fileManager.processFrontMatter(file, (fm) => {
            fm["visit-count"] = visitCount;
            fm["confidence-level"] = confidence;
            fm["consecutive-correct"] = streak;
            fm["last-reviewed"] = new Date().toISOString().split('T')[0];
            if (index > 0) fm["last-struggle-date"] = new Date().toISOString().split('T')[0];
        });
        
        status.innerHTML = `
            <strong>Learning Progress</strong><br>
            Reviews: ${visitCount} | Confidence: ${confidence}/5 | Streak: ${streak}
        `;
    });
});
```

```dataviewjs
// Get all flashcards from the current note
const currentPage = dv.current();
const content = await app.vault.read(app.vault.getAbstractFileByPath(currentPage.file.path));

// Split content into lines
const lines = content.split('\n');
let flashcardLines = [];
let inCodeBlock = false;

// Collect all potential flashcard lines - simplified approach
for (let i = 0; i < lines.length; i++) {
    const line = lines[i];
    
    // Track code blocks
    if (line.trim().startsWith('```')) {
        inCodeBlock = !inCodeBlock;
        continue;
    }
    
    // Skip everything inside code blocks
    if (inCodeBlock) continue;
    
    // Check if line contains flashcard separator
    if (line.includes(';;')) {
        flashcardLines.push(line);
    }
}

const filteredLines = flashcardLines.filter(line => {
    // Only filter out lines that are clearly part of the JavaScript code
    // Be more specific with patterns to avoid false positives
    return !(
        line.trim().startsWith('const ') ||
        line.trim().startsWith('let ') ||
        line.trim().startsWith('function ') ||
        line.trim().startsWith('return ') ||
        line.trim().startsWith('if (') ||
        line.trim().startsWith('for (') ||
        line.trim().startsWith('while (') ||
        line.includes('dataviewjs') ||
        line.includes('content.split') ||
        line.includes('flashcardLines') ||
        line.includes('this.container') ||
        line.includes('addEventListener') ||
        line.includes('console.log') ||
        /\.\w+\(/.test(line) && (line.includes('.map(') || line.includes('.filter(') || line.includes('.forEach(') || line.includes('.find('))
    );
});

// Process the flashcards
const flashcards = [];
for (let i = 0; i < filteredLines.length; i++) {
    const line = filteredLines[i];
    try {
        const separatorIndex = line.indexOf(';;');
        if (separatorIndex === -1) continue;
        
        const front = line.substring(0, separatorIndex).trim();
        const back = line.substring(separatorIndex + 2).trim();
        
        // Very minimal validation - just check they exist
        if (front && back) {
            flashcards.push({ 
                front: front, 
                back: back,
                index: i // Keep track of original order
            });
        }
    } catch (error) {
        console.log('Error processing flashcard line:', line, error);
    }
}

// Debug information
console.log('All lines with ;;:', flashcardLines.length);
console.log('After filtering:', filteredLines.length);
console.log('Filtered lines:', filteredLines);
console.log('Final flashcards:', flashcards.length);

if (flashcards.length === 0) {
    const errorMsg = this.container.createEl('div');
    errorMsg.style.cssText = 'background: #2a2a2a; padding: 15px; border-radius: 6px; color: #cccccc;';
    errorMsg.innerHTML = `
        <strong>No flashcards found!</strong><br><br>
        Lines with ';;' found: ${flashcardLines.length}<br>
        After filtering: ${filteredLines.length}<br>
        Valid flashcards: ${flashcards.length}<br><br>
        <strong>All lines with ;;:</strong><br>
        ${flashcardLines.map((line, i) => `${i+1}. ${line.substring(0, 50)}...`).join('<br>')}
    `;
    return;
}

// Flashcard state
let currentCardIndex = 0;
let showingBack = false;
let correctCount = 0;
let totalReviewed = 0;

// Create main container
const container = this.container.createEl('div');
container.style.cssText = `
    background: #2a2a2a;
    border: 1px solid #404040;
    border-radius: 8px;
    padding: 20px;
    margin: 15px 0;
    max-width: 700px;
`;

// Header with progress
const header = container.createEl('div');
header.style.cssText = 'display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px;';

const title = header.createEl('h3');
title.textContent = `Flashcards (${flashcards.length} total)`;
title.style.cssText = 'margin: 0; color: #ffffff;';

const progress = header.createEl('div');
progress.style.cssText = 'color: #cccccc; font-size: 14px; text-align: right;';

// Card container
const cardContainer = container.createEl('div');
cardContainer.style.cssText = `
    background: #1a1a1a;
    border: 2px solid #404040;
    border-radius: 6px;
    padding: 30px;
    margin: 20px 0;
    min-height: 120px;
    display: flex;
    align-items: center;
    justify-content: center;
    text-align: center;
    cursor: pointer;
    transition: all 0.2s;
`;

const cardText = cardContainer.createEl('div');
cardText.style.cssText = `
    font-size: 16px;
    line-height: 1.5;
    color: #ffffff;
    word-wrap: break-word;
    max-width: 100%;
`;

// Button container
const buttonContainer = container.createEl('div');
buttonContainer.style.cssText = 'display: flex; gap: 10px; justify-content: center; flex-wrap: wrap; margin-bottom: 15px;';

// Control buttons
const flipButton = buttonContainer.createEl('button');
flipButton.textContent = 'Flip Card';
flipButton.style.cssText = `
    background: #4a9eff; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px; font-weight: 500;
`;

const easyButton = buttonContainer.createEl('button');
easyButton.textContent = 'Easy ✅';
easyButton.style.cssText = `
    background: #28a745; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const hardButton = buttonContainer.createEl('button');
hardButton.textContent = 'Hard ❌';
hardButton.style.cssText = `
    background: #dc3545; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px; display: none;
`;

const nextButton = buttonContainer.createEl('button');
nextButton.textContent = 'Next →';
nextButton.style.cssText = `
    background: #6c757d; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const prevButton = buttonContainer.createEl('button');
prevButton.textContent = '← Prev';
prevButton.style.cssText = `
    background: #6c757d; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

const shuffleButton = buttonContainer.createEl('button');
shuffleButton.textContent = '🔀 Shuffle';
shuffleButton.style.cssText = `
    background: #17a2b8; color: white; border: none; border-radius: 4px;
    padding: 10px 16px; cursor: pointer; font-size: 14px;
`;

// Functions
function updateDisplay() {
    const card = flashcards[currentCardIndex];
    cardText.textContent = showingBack ? card.back : card.front;
    
    progress.innerHTML = `Card ${currentCardIndex + 1} of ${flashcards.length}`;
    if (totalReviewed > 0) {
        progress.innerHTML += `<br>Correct: ${correctCount}/${totalReviewed} (${Math.round(correctCount/totalReviewed*100)}%)`;
    }
    
    // Show/hide difficulty buttons
    if (showingBack) {
        easyButton.style.display = 'inline-block';
        hardButton.style.display = 'inline-block';
        flipButton.textContent = 'Show Front';
        cardContainer.style.borderColor = '#ffc107';
        cardContainer.style.backgroundColor = '#252525';
    } else {
        easyButton.style.display = 'none';
        hardButton.style.display = 'none';
        flipButton.textContent = 'Flip Card';
        cardContainer.style.borderColor = '#404040';
        cardContainer.style.backgroundColor = '#1a1a1a';
    }
    
    // Update navigation buttons
    prevButton.style.display = currentCardIndex > 0 ? 'inline-block' : 'none';
    nextButton.textContent = currentCardIndex < flashcards.length - 1 ? 'Next →' : 'Restart';
}

function flipCard() {
    showingBack = !showingBack;
    updateDisplay();
}

function nextCard() {
    if (currentCardIndex < flashcards.length - 1) {
        currentCardIndex++;
    } else {
        currentCardIndex = 0;
    }
    showingBack = false;
    updateDisplay();
}

function prevCard() {
    if (currentCardIndex > 0) {
        currentCardIndex--;
        showingBack = false;
        updateDisplay();
    }
}

function markCorrect() {
    if (showingBack) {
        correctCount++;
        totalReviewed++;
        nextCard();
    }
}

function markIncorrect() {
    if (showingBack) {
        totalReviewed++;
        nextCard();
    }
}

function shuffleCards() {
    for (let i = flashcards.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [flashcards[i], flashcards[j]] = [flashcards[j], flashcards[i]];
    }
    currentCardIndex = 0;
    showingBack = false;
    correctCount = 0;
    totalReviewed = 0;
    updateDisplay();
}

// Event listeners
cardContainer.addEventListener('click', flipCard);
flipButton.addEventListener('click', flipCard);
easyButton.addEventListener('click', markCorrect);
hardButton.addEventListener('click', markIncorrect);
nextButton.addEventListener('click', nextCard);
prevButton.addEventListener('click', prevCard);
shuffleButton.addEventListener('click', shuffleCards);

// Instructions
const instructions = container.createEl('div');
instructions.style.cssText = 'font-size: 12px; color: #888; text-align: center; line-height: 1.4;';
instructions.innerHTML = `
    <strong>Controls:</strong> Click card to flip | Navigation buttons | Easy/Hard to mark
`;

// Initialize
updateDisplay();
```
# **Purpose/Why**:
---
The fundamental problem that **Virtual Machines (VMs)** solve is the **inefficient utilization of physical hardware resources** and the **lack of isolation** between different software environments. Traditionally, a single physical server could only run one operating system at a time, leading to wasted CPU, memory, and storage capacity if that OS didn't fully utilize the hardware. Additionally, running multiple applications directly on the same OS could lead to conflicts, security vulnerabilities, and difficulties in managing dependencies.

VMs are incredibly important in computer science for several reasons:

- **Resource Optimization:** ==They allow a single physical machine to run multiple isolated operating systems and applications concurrently,== maximizing hardware utilization and reducing the need for numerous physical servers. This leads to significant cost savings in terms of hardware, power, and cooling.
    
- **Isolation and Security:** Each VM operates in its own isolated environment, meaning that issues or security breaches in one VM generally do not affect others running on the same physical host. This enhances system stability and security.
    
- **Portability:** VMs can be easily moved, copied, and migrated between different physical servers or even different virtualization platforms, making them highly portable. This is crucial for disaster recovery, load balancing, and cloud computing.
    
- **Development and Testing:** Developers can create consistent and isolated environments for testing software across various operating systems and configurations without requiring dedicated physical hardware for each.
    
- **Legacy Applications:** VMs allow organizations to continue running older applications that require specific operating systems or configurations that are no longer supported on modern hardware, extending the life of critical software.
    
- **Cloud Computing Foundation:** VMs are the cornerstone of Infrastructure as a Service (IaaS) cloud offerings, enabling cloud providers to allocate virtualized compute resources to millions of users on demand.
    

# **Core Explanation:**
---
A **Virtual Machine (VM)** is a software-based emulation of a physical computer system. It behaves like a complete, standalone computer, with its own operating system (called the **guest OS**), virtual hardware components (like a CPU, memory, disk drives, network interfaces), and applications. All of this runs on top of a physical host machine and its own operating system (the **host OS**).

Key characteristics of a VM include:

- **Isolation:** Each VM is independent of others running on the same host, and isolated from the host OS. This means a crash or malware in one VM doesn't typically affect others.
    
- **Encapsulation:** A VM is essentially a single file or a set of files on the host's file system, making it easy to save, copy, move, and back up.
    
- **Hardware Independence:** VMs are abstracted from the underlying physical hardware, allowing them to run on different physical machines, provided the virtualization software supports it.
    
- **Resource Sharing:** Multiple VMs share the resources of the single physical host machine, with the virtualization software managing the allocation.
    

Here's how a Virtual Machine works:

At the heart of virtualization is the **Hypervisor** (also known as a Virtual Machine Monitor or VMM). The hypervisor is a layer of software, firmware, or hardware that sits between the physical hardware and the VMs. It is responsible for creating, running, and managing virtual machines.

There are two main types of hypervisors:

1. **Type 1 Hypervisor (Bare-Metal):**
    
    - Installs directly on the physical hardware, without needing an underlying host operating system.
        
    - Examples: VMware ESXi, Microsoft Hyper-V, Citrix XenServer.
        
    - Offers better performance and security because it has direct access to the hardware.
        
    - Often used in data centers and server virtualization.
        
2. **Type 2 Hypervisor (Hosted):**
    
    - Runs as an application on top of a conventional host operating system (e.g., Windows, macOS, Linux).
        
    - Examples: VMware Workstation, Oracle VirtualBox.
        
    - Easier to set up and use for individual users or development environments.
        
    - Performance might be slightly less than Type 1 due to the overhead of the host OS.
        

Regardless of the type, the hypervisor performs the following key functions:

- **Hardware Virtualization:** It virtualizes the physical hardware components (CPU, memory, storage, network) and presents them to each VM as virtual devices.
    
- **Resource Management:** It allocates and schedules the host's physical resources among the running VMs, ensuring that each VM gets the necessary resources without interfering with others.
    
- **Instruction Translation/Emulation:** For guest OSes that are unaware they are running virtually (non-paravirtualized), the hypervisor intercepts and translates sensitive instructions from the guest OS to the host's hardware, ensuring proper execution and isolation. For paravirtualized guests, the guest OS is modified to communicate directly with the hypervisor, improving performance.
    

When a VM starts, the hypervisor loads its virtual disk image and presents its virtual hardware to the guest OS. The guest OS then boots up just as it would on a physical machine, unaware that it's running in a virtualized environment.

# **Related Concepts:**
---
- **Containerization (e.g., Docker, Kubernetes):** While VMs virtualize the entire operating system and hardware, containers virtualize at the operating system level, sharing the host OS kernel. Containers are much lighter-weight and faster to start than VMs, offering greater portability and efficiency for applications, but they provide less isolation than VMs. VMs are like separate houses, while containers are like separate apartments in the same building.
    
- **Cloud Computing (IaaS, PaaS, SaaS):** VMs are the foundational technology for Infrastructure as a Service (IaaS), where users rent virtualized compute resources (VMs) from a cloud provider. Platform as a Service (PaaS) and Software as a Service (SaaS) build upon this, abstracting away even more of the underlying infrastructure, but often still rely on VMs (and containers) beneath the surface.
    
- **Operating System (OS):** A VM requires a guest OS to run inside it, just like a physical computer. The VM provides the virtual "hardware" environment for the guest OS to operate.
    
- **Emulation vs. Virtualization:** Emulation involves one system imitating another, typically by translating instructions for a different architecture (e.g., running an ARM-compiled game on an x86 PC). Virtualization, particularly hardware-assisted virtualization, allows a guest OS to run directly on the CPU (or near-directly) of the same architecture, making it significantly faster than pure emulation. VMs typically use virtualization, not full emulation, for performance.
    
- **Server Consolidation:** This is a key benefit and application of VMs. Instead of having many underutilized physical servers, VMs allow for consolidating multiple workloads onto fewer, more powerful physical machines, leading to reduced hardware costs, power consumption, and data center space.
    

# **Examples:**
---
Directly demonstrating Virtual Machines through "code examples" in a traditional programming sense is not feasible, as VMs are entire software systems managed by hypervisors. However, we can illustrate how one might _interact_ with a hypervisor (e.g., using a command-line tool or a Python library for a virtualization platform) to create or manage a VM.

For this example, we'll imagine interacting with `virt-manager` or `virsh` (command-line tool for KVM/QEMU on Linux), which allows programmatic control over VMs. This isn't "code" in the sense of an application, but rather commands that a system administrator or automation script would execute to manage VMs.

**Example 1: Using `virsh` (KVM/QEMU Command-Line Tool) to define and start a VM**
```bash
# This is a bash script, not a Python program. It simulates
# common commands used to manage Virtual Machines on a Linux host
# with KVM/QEMU virtualization via the 'libvirt' daemon and 'virsh' CLI.

# Note: These commands require a Linux host with KVM/QEMU and libvirt installed
# and proper permissions to manage virtual machines.

# Step 1: Define the VM's configuration using an XML file.
# This XML file describes the virtual hardware (CPU, RAM, disk, network)
# for the new virtual machine.
# In a real scenario, you would create 'my_vm.xml' with desired specs.
# For simplicity, we'll just show the command to define it.
echo "Defining VM from XML configuration..."
# Example content of 'my_vm.xml' (simplified):
# <domain type='kvm'>
#   <name>my_test_vm</name>
#   <memory unit='MiB'>2048</memory>
#   <currentMemory unit='MiB'>2048</currentMemory>
#   <vcpu placement='static'>2</vcpu>
#   <os>
#     <type arch='x86_64' machine='pc-q35-7.2'>hvm</type>
#     <boot dev='hd'/>
#   </os>
#   <devices>
#     <disk type='file' device='disk'>
#       <driver name='qemu' type='qcow2'/>
#       <source file='/var/lib/libvirt/images/my_test_vm.qcow2'/>
#       <target dev='vda' bus='virtio'/>
#     </disk>
#     <interface type='network'>
#       <source network='default'/>
#       <model type='virtio'/>
#     </interface>
#   </devices>
# </domain>

# Define the VM using the XML configuration. This registers the VM with libvirt
# but does not start it.
virsh define my_vm.xml
echo "VM 'my_test_vm' defined."

# Step 2: Start the defined VM.
# This command powers on the virtual machine, allowing its guest OS to boot.
echo "Starting VM 'my_test_vm'..."
virsh start my_test_vm
echo "VM 'my_test_vm' started."

# Step 3: Check the status of the VM.
# This confirms if the VM is running.
echo "Checking VM status..."
virsh list --all
# Expected output will show 'my_test_vm' as 'running'

# Step 4: Connect to the VM's console (optional, for interaction)
# This command allows you to interact with the guest OS's console,
# similar to plugging a monitor and keyboard into a physical machine.
# This might open a separate window or connect via SSH if configured.
# echo "Connecting to VM console (Ctrl+] to exit)..."
# virsh console my_test_vm

# Step 5: Shut down the VM (gracefully).
# This initiates a graceful shutdown within the guest OS, if ACPI is supported.
echo "Shutting down VM 'my_test_vm' gracefully..."
virsh shutdown my_test_vm
# Wait for a moment for the VM to shut down.
sleep 10
echo "VM 'my_test_vm' shutdown initiated."

# Step 6: Destroy (force stop) the VM if it doesn't shut down gracefully.
# This is equivalent to pulling the power plug on a physical machine.
echo "Force stopping VM 'my_test_vm' (if still running)..."
virsh destroy my_test_vm
echo "VM 'my_test_vm' destroyed (stopped)."

# Step 7: Undefine the VM (remove its configuration from libvirt).
# This removes the VM's definition from the hypervisor, but does not delete
# its disk image file.
echo "Undefining VM 'my_test_vm'..."
virsh undefine my_test_vm
echo "VM 'my_test_vm' undefined."

# This script demonstrates the lifecycle of a VM from the perspective of a hypervisor management tool.
# It highlights how VMs are distinct entities that can be defined, started, stopped, and removed,
# all managed by the hypervisor layer that virtualizes the underlying physical hardware.
```

# **Flashcards:**

---

What is a Virtual Machine (VM)?;; A software-based emulation of a complete physical computer system, including its own operating system and virtual hardware, running on a physical host machine. 

What is a Hypervisor?;; A layer of software, firmware, or hardware that creates, runs, and manages virtual machines by abstracting and allocating the physical hardware resources. 

What are the two main types of hypervisors?;; Type 1 (Bare-Metal), which runs directly on hardware (e.g., VMware ESXi), and Type 2 (Hosted), which runs as an application on a host OS (e.g., VirtualBox).