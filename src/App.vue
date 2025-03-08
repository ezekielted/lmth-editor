```vue type="vue" project="HTML Editor" file="app.vue"
[v0-no-op-code-block-prefix]<template>
  <div class="html-editor" :class="{ 'dark': isDarkMode }">
    <div class="app-header">
      <div class="app-title-container">
        <h1 class="app-title">
          <span class="bracket">&lt;</span>
          <span class="letter">l</span>
          <span class="letter">m</span>
          <span class="letter">t</span>
          <span class="letter-h">h</span>
          <span class="bracket">&gt;</span>
        </h1>
        <p class="app-description">Code & Render in Sync</p>
      </div>
      <button @click="toggleDarkMode" class="theme-toggle" aria-label="Toggle dark mode">
        <SunIcon v-if="isDarkMode" class="icon" />
        <MoonIcon v-else class="icon" />
      </button>
    </div>
    
    <div class="toolbar">
      <div class="toolbar-section file-actions">
        <div class="file-input">
          <label for="html-file" class="btn btn-primary">
            <i class="icon"><UploadIcon /></i>
            Upload HTML
          </label>
          <input 
            type="file" 
            id="html-file" 
            accept=".html" 
            @change="handleFileUpload" 
            class="hidden"
          />
        </div>
        
        <button @click="saveHtml" class="btn btn-primary">
          <i class="icon"><SaveIcon /></i> Save
        </button>
        
        <button @click="saveHtmlAs" class="btn btn-primary">
          <i class="icon"><SaveIcon /></i> Save As
        </button>
      </div>
      
      <div class="toolbar-section edit-actions">
        <button @click="undo" class="btn btn-secondary" :disabled="!canUndo">
          <i class="icon"><UndoIcon /></i> Undo
        </button>
        
        <button @click="redo" class="btn btn-secondary" :disabled="!canRedo">
          <i class="icon"><RedoIcon /></i> Redo
        </button>
      </div>
    </div>

    <div class="main-content">
      <div class="panel editor-panel">
        <div class="panel-header">
          <h2 class="panel-title">Visual Editor</h2>
        </div>
        <div class="editor-container">
          <div 
            ref="editorRef" 
            class="editor-content" 
            contenteditable="true"
            @input="handleEditorInput"
            @keydown="handleKeyDown"
            @click="handleEditorClick"
            @mouseup="handleTextSelection"
            @keyup="handleTextSelection"
            @focus="handleEditorFocus"
            @blur="handleEditorBlur"
            @mousedown="handleEditorMouseDown"
          ></div>
          
          <!-- Floating Image Bubble -->
          <div v-if="showImageBubble" class="image-bubble" :style="bubbleStyle">
            <button @click.stop="openImagePrompt" class="bubble-btn">
              <i class="icon"><ImageIcon /></i>
            </button>
          </div>
          
          <!-- Floating Link Bubble -->
          <div v-if="showLinkBubble" class="link-bubble" :style="linkBubbleStyle">
            <button @click.stop="openLinkPrompt" class="bubble-btn">
              <i class="icon"><LinkIcon /></i>
            </button>
          </div>
          
          <!-- Image URL Modal -->
          <div v-if="showImagePrompt" class="image-modal" @click.stop>
            <div class="modal-content">
              <h3 class="modal-title">Insert Image</h3>
              <input 
                v-model="imageUrl" 
                type="text" 
                placeholder="Enter image URL or Google Drive link" 
                @keyup.enter="insertImage"
                ref="imageUrlInput"
                class="modal-input"
              />
              <div class="modal-actions">
                <button @click="cancelImageInsert" class="btn btn-danger">Cancel</button>
                <button @click="insertImage" class="btn btn-primary">Insert</button>
              </div>
            </div>
          </div>
          
          <!-- Link URL Modal -->
          <div v-if="showLinkPrompt" class="image-modal" @click.stop>
            <div class="modal-content">
              <h3 class="modal-title">Insert Link</h3>
              <input 
                v-model="linkUrl" 
                type="text" 
                placeholder="Enter URL (e.g., https://example.com)" 
                @keyup.enter="insertLink"
                ref="linkUrlInput"
                class="modal-input"
              />
              <div class="modal-actions">
                <button @click="cancelLinkInsert" class="btn btn-danger">Cancel</button>
                <button @click="insertLink" class="btn btn-primary">Insert</button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="panel code-panel">
        <div class="panel-header">
          <h2 class="panel-title">HTML Code</h2>
        </div>
        <textarea 
          ref="htmlOutputRef" 
          v-model="currentHtml"
          @input="handleHtmlOutputInput"
          @focus="handleHtmlCodeFocus"
          class="html-output"
        ></textarea>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';
import { 
  ImageIcon, 
  SaveIcon, 
  UndoIcon, 
  RedoIcon, 
  SunIcon, 
  MoonIcon,
  UploadIcon,
  LinkIcon
} from 'lucide-vue-next';

const editorRef = ref(null);
const htmlOutputRef = ref(null);
const currentHtml = ref('<p>Click to edit this text.</p>');
const isDarkMode = ref(false);

// Undo/Redo history
const history = ref([]);
const historyIndex = ref(-1);
const canUndo = ref(false);
const canRedo = ref(false);
const isUndoRedoOperation = ref(false);

// Image insertion
const showImageBubble = ref(false);
const showImagePrompt = ref(false);
const imageUrl = ref('');
const bubbleStyle = ref({});
const imageUrlInput = ref(null);
const savedSelection = ref(null);

// Link insertion
const showLinkBubble = ref(false);
const showLinkPrompt = ref(false);
const linkUrl = ref('');
const linkBubbleStyle = ref({});
const linkUrlInput = ref(null);
const hasTextSelection = ref(false);

// Editor state tracking
const isEditorActive = ref(false);
const isHtmlCodeActive = ref(false);
const lastActiveElement = ref(null);

// Toggle dark mode
const toggleDarkMode = () => {
  isDarkMode.value = !isDarkMode.value;
  
  // Save preference to localStorage
  localStorage.setItem('htmlEditorDarkMode', isDarkMode.value ? 'dark' : 'light');
  
  // Update HTML code section styles based on dark mode
  updateHtmlCodeStyles();
};

// Update HTML code section styles based on dark mode
const updateHtmlCodeStyles = () => {
    if (htmlOutputRef.value) {
      if (isDarkMode.value) {
        htmlOutputRef.value.style.backgroundColor = '#1e293b';
        htmlOutputRef.value.style.color = '#f1f5f9';
      } else {
        htmlOutputRef.value.style.backgroundColor = 'white';
        htmlOutputRef.value.style.color = '#1e293b';
      }
    }
    
    // Visual Editor always stays white with dark text for readability
    if (editorRef.value) {
      editorRef.value.style.backgroundColor = 'white';
      editorRef.value.style.color = '#1e293b';
    }
  };

// Save the current selection
const saveSelection = () => {
  if (window.getSelection) {
    const sel = window.getSelection();
    if (sel.getRangeAt && sel.rangeCount) {
      savedSelection.value = sel.getRangeAt(0);
    }
  }
};

// Restore the saved selection
const restoreSelection = () => {
  if (savedSelection.value) {
    if (window.getSelection) {
      const sel = window.getSelection();
      sel.removeAllRanges();
      sel.addRange(savedSelection.value);
    }
  }
};

// Save the current scroll position of the editor
const saveEditorScrollPosition = () => {
  if (editorRef.value) {
    return editorRef.value.parentElement.scrollTop;
  }
  return 0;
};

// Restore the editor scroll position
const restoreEditorScrollPosition = (position) => {
  if (editorRef.value) {
    editorRef.value.parentElement.scrollTop = position;
  }
};

// Handle file upload
const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  const reader = new FileReader();
  reader.onload = (e) => {
    setHtml(e.target.result);
  };
  reader.readAsText(file);
};

// Function to extract Google Drive image ID
const extractGoogleDriveImageId = (url) => {
  const regex = /\/file\/d\/([a-zA-Z0-9_-]+)/;
  const match = url.match(regex);
  return match ? match[1] : null;
};

// Function to check if a string is a valid URL
const isValidUrl = (string) => {
  try {
    new URL(string);
    return true;
  } catch (_) {
    return false;
  }
};

// Set HTML content
const setHtml = (html) => {
  if (editorRef.value) {
    editorRef.value.innerHTML = html;
    currentHtml.value = html;
    addToHistory(html);
  }
};

// Handle input events from the editor
const handleEditorInput = () => {
  if (editorRef.value) {
    const newHtml = editorRef.value.innerHTML;
    if (newHtml !== currentHtml.value) {
      currentHtml.value = newHtml;
      
      // Only add to history if this is not an undo/redo operation
      if (!isUndoRedoOperation.value) {
        addToHistory(newHtml);
      }
    }
  }
};

// Handle input events from the HTML output
const handleHtmlOutputInput = () => {
    if (htmlOutputRef.value) {
      try {
        // Update the editor with the new HTML
        editorRef.value.innerHTML = currentHtml.value;
        
        // Only add to history if this is not an undo/redo operation
        if (!isUndoRedoOperation.value) {
          addToHistory(currentHtml.value);
        }
      } catch (error) {
        console.error('Invalid HTML:', error);
      }
    }
  };

// Handle focus on the editor
const handleEditorFocus = () => {
  isEditorActive.value = true;
  isHtmlCodeActive.value = false;
};

// Handle blur on the editor
const handleEditorBlur = () => {
  isEditorActive.value = false;
};

// Handle focus on the HTML code
const handleHtmlCodeFocus = () => {
  isHtmlCodeActive.value = true;
  isEditorActive.value = false;
};

// Handle mouse down in the editor to track the active element
const handleEditorMouseDown = () => {
  // Wait for the selection to be updated
  setTimeout(() => {
    const selection = window.getSelection();
    if (selection && selection.rangeCount > 0) {
      const range = selection.getRangeAt(0);
      let element = range.commonAncestorContainer;
      
      // If we clicked on a text node, get its parent element
      if (element.nodeType === 3) {
        element = element.parentNode;
      }
      
      // Store the active element
      lastActiveElement.value = element;
    }
  }, 0);
};

// Handle text selection in the editor
const handleTextSelection = () => {
  const selection = window.getSelection();
  
  // Hide link bubble if there's no selection
  if (!selection || selection.isCollapsed || !selection.toString().trim()) {
    showLinkBubble.value = false;
    hasTextSelection.value = false;
    return;
  }
  
  // Check if selection is within the editor
  let isInEditor = false;
  let node = selection.anchorNode;
  while (node != null) {
    if (node === editorRef.value) {
      isInEditor = true;
      break;
    }
    node = node.parentNode;
  }
  
  if (!isInEditor) {
    showLinkBubble.value = false;
    hasTextSelection.value = false;
    return;
  }
  
  // We have a valid text selection in the editor
  hasTextSelection.value = true;
  saveSelection();
  
  // Save the current scroll position (for potential future use)
  // eslint-disable-next-line no-unused-vars
  const scrollPosition = saveEditorScrollPosition();
  
  // Position the link bubble near the selection
  const range = selection.getRangeAt(0);
  const rect = range.getBoundingClientRect();
  const editorRect = editorRef.value.getBoundingClientRect();
  
  linkBubbleStyle.value = {
    left: `${rect.left - editorRect.left + rect.width / 2}px`,
    top: `${rect.top - editorRect.top - 30}px`
  };
  
  showLinkBubble.value = true;
  
  // Hide bubble after a delay if not interacted with
  setTimeout(() => {
    if (!showLinkPrompt.value) {
      showLinkBubble.value = false;
    }
  }, 3000);
};

// Handle keydown events
const handleKeyDown = (event) => {
  // Handle Undo (Ctrl+Z)
  if (event.ctrlKey && event.key === 'z') {
    event.preventDefault();
    undo();
    return;
  }
  
  // Handle Redo (Ctrl+Y)
  if (event.ctrlKey && event.key === 'y') {
    event.preventDefault();
    redo();
    return;
  }
  
  // If Enter key is pressed, create a new element
  if (event.key === 'Enter') {
    event.preventDefault();
    
    // Get the current selection
    const selection = window.getSelection();
    if (!selection.rangeCount) return;
    
    const range = selection.getRangeAt(0);
    
    // Get the current parent element
    let currentElement = range.commonAncestorContainer;
    if (currentElement.nodeType === 3) { // Text node
      currentElement = currentElement.parentNode;
    }
    
    // Get the tag name of the current element
    const tagName = currentElement.tagName.toLowerCase();
    
    // Only handle specific text elements
    const textElements = ['p', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'div', 'span'];
    const newTagName = textElements.includes(tagName) ? tagName : 'p';
    
    // Split the content at the cursor position
    const currentContent = currentElement.innerHTML;
    const cursorOffset = range.startOffset;
    
    // If we're in a text node, we need to find the offset within the parent's HTML
    let totalOffset = cursorOffset;
    if (range.startContainer.nodeType === 3) { // Text node
      let node = currentElement.firstChild;
      while (node && node !== range.startContainer) {
        if (node.nodeType === 3) { // Text node
          totalOffset += node.length;
        } else {
          totalOffset += node.outerHTML.length;
        }
        node = node.nextSibling;
      }
    }
    
    // Split the content
    const beforeCursor = currentContent.substring(0, totalOffset);
    const afterCursor = currentContent.substring(totalOffset);
    
    // Update the current element with content before cursor
    currentElement.innerHTML = beforeCursor || '&nbsp;';
    
    // Create a new element with content after cursor
    const newElement = document.createElement(newTagName);
    newElement.innerHTML = afterCursor || '&nbsp;';
    
    // Insert the new element after the current one
    if (currentElement.nextSibling) {
      currentElement.parentNode.insertBefore(newElement, currentElement.nextSibling);
    } else {
      currentElement.parentNode.appendChild(newElement);
    }
    
    // Set cursor to the beginning of the new element
    const newRange = document.createRange();
    const newTextNode = newElement.firstChild;
    
    if (newTextNode) {
      if (newTextNode.nodeType === 3) { // Text node
        newRange.setStart(newTextNode, 0);
        newRange.setEnd(newTextNode, 0);
      } else {
        newRange.setStart(newElement, 0);
        newRange.setEnd(newElement, 0);
      }
      
      selection.removeAllRanges();
      selection.addRange(newRange);
    }
    
    handleEditorInput();
    
    // Update the active element
    lastActiveElement.value = newElement;
  }
};

// Handle click inside the editor
const handleEditorClick = (event) => {
  const x = event.clientX;
  const y = event.clientY;
  
  // Save the current selection
  saveSelection();
  
  // Position the bubble relative to the editor container
  const rect = editorRef.value.getBoundingClientRect();
  bubbleStyle.value = {
    left: `${x - rect.left + 10}px`,
    top: `${y - rect.top - 30}px`
  };
  
  showImageBubble.value = true;
  
  // Hide bubble after a delay if not interacted with
  setTimeout(() => {
    if (!showImagePrompt.value) {
      showImageBubble.value = false;
    }
  }, 3000);
};

// Open the image URL prompt
const openImagePrompt = () => {
  showImageBubble.value = false;
  showImagePrompt.value = true;
  
  setTimeout(() => {
    if (imageUrlInput.value) {
      imageUrlInput.value.focus();
    }
  }, 0);
};

// Open the link URL prompt
const openLinkPrompt = () => {
  showLinkBubble.value = false;
  showLinkPrompt.value = true;
  
  setTimeout(() => {
    if (linkUrlInput.value) {
      linkUrlInput.value.focus();
    }
  }, 0);
};

// Cancel image insertion
const cancelImageInsert = () => {
  showImagePrompt.value = false;
  imageUrl.value = '';
};

// Cancel link insertion
const cancelLinkInsert = () => {
  showLinkPrompt.value = false;
  linkUrl.value = '';
  
  // Restore the editor scroll position
  const scrollPosition = saveEditorScrollPosition();
  restoreEditorScrollPosition(scrollPosition);
};

// Insert image at current position
const insertImage = () => {
  if (!imageUrl.value) {
    showImagePrompt.value = false;
    return;
  }
  
  let imgSrc;
  const googleDriveId = extractGoogleDriveImageId(imageUrl.value);
  
  if (googleDriveId) {
    imgSrc = `https://drive.google.com/thumbnail?id=${googleDriveId}&sz=s4000`;
  } else if (isValidUrl(imageUrl.value)) {
    imgSrc = imageUrl.value;
  } else {
    alert('Invalid image URL. Please enter a valid URL or Google Drive link.');
    return;
  }
  
  // Save the current scroll position
  const scrollPosition = saveEditorScrollPosition();
  
  // Create an image element with proper styling
  const img = document.createElement('img');
  img.src = imgSrc;
  img.alt = 'Inserted image';
  img.style.maxWidth = '100%';
  img.style.height = 'auto';
  img.className = 'editor-image'; // Add a class for additional styling
  
  // Focus the editor and restore selection before inserting
  editorRef.value.focus();
  restoreSelection();
  
  // Insert the image as a DOM element instead of HTML string
  const selection = window.getSelection();
  if (selection.rangeCount) {
    const range = selection.getRangeAt(0);
    range.deleteContents();
    range.insertNode(img);
    
    // Move cursor after the image
    range.setStartAfter(img);
    range.setEndAfter(img);
    selection.removeAllRanges();
    selection.addRange(range);
  }
  
  // Update HTML
  handleEditorInput();
  imageUrl.value = '';
  showImagePrompt.value = false;
  
  // Restore the scroll position
  nextTick(() => {
    restoreEditorScrollPosition(scrollPosition);
    
    // Update the active element
    lastActiveElement.value = img;
  });
};

// Insert link for selected text
const insertLink = () => {
  if (!linkUrl.value) {
    showLinkPrompt.value = false;
    const scrollPosition = saveEditorScrollPosition();
    restoreEditorScrollPosition(scrollPosition);
    return;
  }
  
  // Save the current scroll position
  const scrollPosition = saveEditorScrollPosition();
  
  // Validate URL
  let href = linkUrl.value;
  if (!href.startsWith('http://') && !href.startsWith('https://') && !href.startsWith('mailto:') && !href.startsWith('#')) {
    href = 'https://' + href;
  }
  
  if (!isValidUrl(href) && !href.startsWith('mailto:') && !href.startsWith('#')) {
    alert('Please enter a valid URL.');
    return;
  }
  
  // Focus the editor and restore selection before inserting
  editorRef.value.focus();
  restoreSelection();
  
  // Get the selected text
  const selection = window.getSelection();
  if (selection.rangeCount) {
    const range = selection.getRangeAt(0);
    const selectedText = range.toString();
    
    // Create an anchor element
    const anchor = document.createElement('a');
    anchor.href = href;
    anchor.textContent = selectedText;
    anchor.target = '_blank'; // Open in new tab
    anchor.rel = 'noopener noreferrer'; // Security best practice
    
    // Replace the selected text with the anchor
    range.deleteContents();
    range.insertNode(anchor);
    
    // Move cursor after the link
    range.setStartAfter(anchor);
    range.setEndAfter(anchor);
    selection.removeAllRanges();
    selection.addRange(range);
    
    // Update the active element
    lastActiveElement.value = anchor;
  }
  
  // Update HTML
  handleEditorInput();
  linkUrl.value = '';
  showLinkPrompt.value = false;
  
  // Restore the scroll position
  nextTick(() => {
    restoreEditorScrollPosition(scrollPosition);
  });
};

// Add current HTML to history
const addToHistory = (html) => {
  // If we're not at the end of the history, truncate it
  if (historyIndex.value < history.value.length - 1) {
    history.value = history.value.slice(0, historyIndex.value + 1);
  }
  
  history.value.push(html);
  historyIndex.value = history.value.length - 1;
  updateUndoRedoState();
};

// Update undo/redo button states
const updateUndoRedoState = () => {
  canUndo.value = historyIndex.value > 0;
  canRedo.value = historyIndex.value < history.value.length - 1;
};

// Undo the last change
const undo = () => {
  if (historyIndex.value > 0) {
    isUndoRedoOperation.value = true;
    historyIndex.value--;
    
    // Update the editor with the previous state
    if (editorRef.value) {
      editorRef.value.innerHTML = history.value[historyIndex.value];
      currentHtml.value = history.value[historyIndex.value];
    }
    
    updateUndoRedoState();
    isUndoRedoOperation.value = false;
  }
};

// Redo the last undone change
const redo = () => {
  if (historyIndex.value < history.value.length - 1) {
    isUndoRedoOperation.value = true;
    historyIndex.value++;
    
    // Update the editor with the next state
    if (editorRef.value) {
      editorRef.value.innerHTML = history.value[historyIndex.value];
      currentHtml.value = history.value[historyIndex.value];
    }
    
    updateUndoRedoState();
    isUndoRedoOperation.value = false;
  }
};

// Save the HTML to a file
const saveHtml = () => {
  const blob = new Blob([currentHtml.value], { type: 'text/html' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'document.html';
  a.click();
  URL.revokeObjectURL(url);
};

// Save the HTML to a file with a custom name
const saveHtmlAs = () => {
  const fileName = prompt('Enter file name:', 'document.html');
  if (!fileName) return;
  
  const blob = new Blob([currentHtml.value], { type: 'text/html' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = fileName.endsWith('.html') ? fileName : `${fileName}.html`;
  a.click();
  URL.revokeObjectURL(url);
};

// Initialize the editor
onMounted(() => {
  // Check for saved dark mode preference
  const savedMode = localStorage.getItem('htmlEditorDarkMode');
  if (savedMode === 'dark') {
    isDarkMode.value = true;
  }

  // Apply initial styles
  updateHtmlCodeStyles();
  
  // Set initial HTML content
  if (editorRef.value) {
    editorRef.value.innerHTML = currentHtml.value;
    addToHistory(currentHtml.value);
  }
});
</script>

<style scoped>
/* Base styles and variables */
.html-editor {
  --primary-color: #2d3748;
  --primary-hover: #1a202c;
  --secondary-color: #4a5568;
  --secondary-hover: #2d3748;
  --danger-color: #e53e3e;
  --danger-hover: #c53030;
  --text-color: #1e293b;
  --text-secondary: #475569;
  --bg-color: #ffffff;
  --bg-secondary: #f8fafc;
  --border-color: #e2e8f0;
  --shadow-color: rgba(0, 0, 0, 0.1);
  --scrollbar-bg: #f1f5f9;
  --scrollbar-thumb: #94a3b8;
  --bracket-color: var(--text-color);
  --h-bg-color: #000000;
  --h-text-color: #ffffff;

  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  
  /* Fullscreen modifications - remove constraints */
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
  
  /* Use flexbox for better layout control */
  display: flex;
  flex-direction: column;
  
  color: var(--text-color);
  background-color: var(--bg-color);
  transition: all 0.3s ease;
  
  /* Make sure it takes the full screen */
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  overflow: hidden;
  box-sizing: border-box;
}

/* Dark mode styles */
.html-editor.dark {
  --primary-color: #1a202c;
  --primary-hover: #0f172a;
  --secondary-color: #4a5568;
  --secondary-hover: #2d3748;
  --danger-color: #f87171;
  --danger-hover: #ef4444;
  --text-color: #f1f5f9;
  --text-secondary: #cbd5e1;
  --bg-color: #0f172a;
  --bg-secondary: #1e293b;
  --border-color: #334155;
  --shadow-color: rgba(0, 0, 0, 0.5);
  --scrollbar-bg: #1e293b;
  --scrollbar-thumb: #64748b;
  --bracket-color: var(--text-color);
  --h-bg-color: #ffffff;
  --h-text-color: #000000;
}

/* App header - Improved for better responsiveness */
.app-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 15px;
  border-bottom: 2px solid var(--border-color);
  width: 100%;
  flex-shrink: 0;
  min-height: 60px;
  box-sizing: border-box;
}

.app-title-container {
  display: flex;
  flex-direction: column;
  flex-shrink: 1;
  overflow: hidden;
  margin-right: 10px;
}

.app-title {
  font-size: clamp(1.5rem, 4vw, 2rem);
  font-weight: 700;
  margin: 0;
  display: flex;
  align-items: center;
  font-family: 'Pacifico', 'Comic Sans MS', cursive;
  letter-spacing: 1px;
  white-space: nowrap;
}

.bracket {
  color: var(--bracket-color);
  font-weight: 500;
}

.letter {
  color: var(--text-color);
  margin: 0 1px;
}

.letter-h {
  color: var(--h-text-color);
  background-color: var(--h-bg-color);
  padding: 0 5px;
  border-radius: 4px;
  margin: 0 1px;
}

.app-description {
  font-size: clamp(0.75rem, 2vw, 0.95rem);
  color: var(--text-secondary);
  margin: 5px 0 0 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.theme-toggle {
  background: none;
  border: none;
  color: var(--text-color);
  cursor: pointer;
  width: 40px;
  height: 40px;
  min-width: 40px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background-color 0.2s;
  flex-shrink: 0;
  margin-left: auto;
}

.theme-toggle:hover {
  background-color: var(--bg-secondary);
}

/* Toolbar - Improved for better responsiveness */
.toolbar {
  display: flex;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 10px;
  background-color: var(--bg-secondary);
  padding: 10px 15px;
  box-shadow: 0 2px 4px var(--shadow-color);
  flex-shrink: 0;
  z-index: 1;
  box-sizing: border-box;
}

.toolbar-section {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  align-items: center;
}

/* Buttons - Improved for better responsiveness */
.btn {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 12px;
  border-radius: 6px;
  font-weight: 500;
  font-size: 0.85rem;
  cursor: pointer;
  transition: all 0.2s;
  border: none;
  white-space: nowrap;
}

@media (max-width: 480px) {
  .btn {
    padding: 6px 10px;
    font-size: 0.8rem;
  }
  
  .icon {
    width: 16px;
    height: 16px;
  }
}

.btn-primary {
  background-color: var(--primary-color);
  color: white;
}

.btn-primary:hover {
  background-color: var(--primary-hover);
}

.btn-secondary {
  background-color: var(--secondary-color);
  color: white;
}

.btn-secondary:hover {
  background-color: var(--secondary-hover);
}

.btn-danger {
  background-color: var(--danger-color);
  color: white;
}

.btn-danger:hover {
  background-color: var(--danger-hover);
}

.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.hidden {
  display: none;
}

/* Main content layout - now with flex-grow to fill available space */
.main-content {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 15px;
  padding: 15px;
  flex-grow: 1;
  overflow: hidden;
  box-sizing: border-box;
}

@media (max-width: 768px) {
  .main-content {
    grid-template-columns: 1fr;
    gap: 10px;
    padding: 10px;
  }
}

/* Panels - modify to use full height */
.panel {
  background-color: var(--bg-secondary);
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 4px var(--shadow-color);
  display: flex;
  flex-direction: column;
  height: 100%;
}

.panel-header {
  background-color: var(--primary-color);
  color: white;
  padding: 10px 15px;
}

.panel-title {
  margin: 0;
  font-size: 1.1rem;
  font-weight: 600;
  color: white; /* Ensure high contrast in both modes */
}

/* Editor - modify to use full height of panel */
.editor-container {
  border: 1px solid var(--border-color);
  border-radius: 0 0 8px 8px;
  padding: 15px;
  position: relative;
  flex-grow: 1;
  overflow: auto;
  background-color: white; /* Always white background */
}

/* Custom scrollbar styles */
.editor-container::-webkit-scrollbar,
.html-output::-webkit-scrollbar {
  width: 10px;
  height: 10px;
}

.editor-container::-webkit-scrollbar-track,
.html-output::-webkit-scrollbar-track {
  background: var(--scrollbar-bg);
}

.editor-container::-webkit-scrollbar-thumb,
.html-output::-webkit-scrollbar-thumb {
  background-color: var(--scrollbar-thumb);
  border-radius: 5px;
  border: 2px solid var(--scrollbar-bg);
}

.editor-content {
  outline: none;
  min-height: 100%;
  color: #1e293b; /* Always dark text for readability */
  background-color: white; /* Always white background */
}

.editor-content img {
  max-width: 100%;
  height: auto;
  display: block;
  margin: 10px 0;
  border-radius: 4px;
}

.editor-content .editor-image {
  max-width: 100%;
  height: auto;
  display: block;
  margin: 10px 0;
  border-radius: 4px;
}

.editor-content a {
  color: #2563eb;
  text-decoration: underline;
}

/* HTML Output - modify to use full height of panel */
.html-output {
  white-space: pre-wrap;
  word-break: break-all;
  font-family: 'Fira Code', monospace;
  font-size: 14px;
  padding: 15px;
  margin: 0;
  outline: none;
  border: none;
  border-radius: 0 0 8px 8px;
  flex-grow: 1;
  overflow: auto;
  text-align: left; /* Ensure HTML code is left-aligned */
  resize: none; /* Prevent textarea resizing */
  width: 100%;
  height: 100%;
  box-sizing: border-box;
  /* Background and text color will be set dynamically in JS */
}

/* Image bubble */
.image-bubble {
  position: absolute;
  z-index: 10;
}

/* Link bubble */
.link-bubble {
  position: absolute;
  z-index: 10;
}

.bubble-btn {
  background-color: var(--primary-color);
  color: white;
  border: none;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  box-shadow: 0 2px 5px var(--shadow-color);
  transition: background-color 0.2s;
}

.bubble-btn:hover {
  background-color: var(--primary-hover);
}

/* Modal */
.image-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 20;
}

.modal-content {
  background-color: var(--bg-color);
  padding: 20px;
  border-radius: 12px;
  width: 90%;
  max-width: 400px;
  box-shadow: 0 4px 20px var(--shadow-color);
}

.modal-title {
  margin-top: 0;
  margin-bottom: 15px;
  font-size: 1.2rem;
  color: var(--text-color);
}

.modal-input {
  width: 100%;
  padding: 10px;
  border: 1px solid var(--border-color);
  border-radius: 6px;
  margin-bottom: 15px;
  background-color: var(--bg-secondary);
  color: var(--text-color);
  font-size: 0.95rem;
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
}

/* Icons */
.icon {
  display: inline-flex;
  align-items: center;
  width: 18px;
  height: 18px;
}
</style>