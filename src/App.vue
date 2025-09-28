<template>
  <div class="html-editor" :class="{ 'dark': isDarkMode }">
    <div class="app-header">
      <div class="app-title-container">
        <h1 class="app-title">
          <button @click="shiftLeft" class="bracket-button" aria-label="Shift letters left">&lt;</button>
          
          <TransitionGroup name="logo" tag="div" class="logo-letters">
            <span 
              v-for="(letter, index) in logoLetters" 
              :key="letter"
              :class="{ 'letter-h': index === logoLetters.length - 1, 'letter': index !== logoLetters.length - 1 }"
            >
              {{ letter }}
            </span>
          </TransitionGroup>

          <button @click="shiftRight" class="bracket-button" aria-label="Shift letters right">&gt;</button>
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

    <div class="main-content" ref="mainContentRef">
      <div class="panel editor-panel" :style="isMobileView ? { height: editorHeight } : { width: editorWidth }">
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
            @mouseover="handleVisualEditorHover"
            @mouseleave="clearHighlight"
            @contextmenu.prevent="handleImageRightClick"
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

          <!-- Image Replace Context Menu -->
          <div v-if="showImageReplaceMenu" class="image-replace-menu" :style="imageReplaceMenuStyle" @click.stop>
            <input 
              v-model="newImageUrl" 
              type="text" 
              placeholder="Enter new image URL"
              @keyup.enter="replaceImage"
            />
            <button @click="replaceImage" class="btn btn-primary">Replace</button>
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

      <div class="resizer" @mousedown="startResize"></div>

      <div class="panel code-panel" :style="isMobileView ? { height: codeHeight } : { width: codeWidth }">
        <div class="panel-header">
          <h2 class="panel-title">HTML Code</h2>
        </div>
        <div class="code-editor-container">
          <pre ref="highlighterRef" class="code-highlighter" aria-hidden="true"></pre>
          <textarea
            ref="htmlOutputRef"
            v-model="currentHtml"
            @input="handleHtmlOutputInput"
            @focus="handleHtmlCodeFocus"
            @scroll="syncScroll"
            class="html-output-textarea"
            spellcheck="false"
          ></textarea>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick, watch } from 'vue';
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

// --- NEW: Logo state ---
const logoLetters = ref(['l', 'm', 't', 'h']);

const editorRef = ref(null);
const htmlOutputRef = ref(null);
const highlighterRef = ref(null);
const currentHtml = ref('<h2>Welcome, friend! Click to edit me</h2><p>Hover to find me 🙂</p>');
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

// Image replacement
const showImageReplaceMenu = ref(false);
const imageReplaceMenuStyle = ref({});
const newImageUrl = ref('');
const targetImage = ref(null);

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

// Highlighting and sync scroll
const isVisualEditorScrolling = ref(false);
let visualScrollTimeout = null;
let highlightDebounceTimer = null;

// --- Resizing panels ---
const mainContentRef = ref(null);
const isDragging = ref(false);
const isMobileView = ref(false); // NEW: State for mobile view detection

// Horizontal (desktop) sizing
const editorWidth = ref('50%');
const codeWidth = ref('50%');

// Vertical (mobile) sizing
const editorHeight = ref('50%');
const codeHeight = ref('50%');

// --- UPDATED: Logo movement functions ---
const shiftRight = () => {
  const currentState = logoLetters.value.join('');

  // From the final state 'html', loop back to the start 'lmth'
  if (currentState === 'html') {
    logoLetters.value = ['l', 'm', 't', 'h'];
  // From the state just before the end, go to 'html'
  } else if (currentState === 'mthl') {
    logoLetters.value = ['h', 't', 'm', 'l'];
  // Otherwise, perform a normal circular shift to the right
  } else {
    const last = logoLetters.value.pop();
    logoLetters.value.unshift(last);
  }
};

const shiftLeft = () => {
  const currentState = logoLetters.value.join('');
  
  // From the start 'lmth', go to the final state 'html'
  if (currentState === 'lmth') {
    logoLetters.value = ['h', 't', 'm', 'l'];
  // From 'html', go to the previous state in the sequence
  } else if (currentState === 'html') {
    logoLetters.value = ['m', 't', 'h', 'l'];
  // Otherwise, perform a normal circular shift to the left
  } else {
    const first = logoLetters.value.shift();
    logoLetters.value.push(first);
  }
};

// Close image replace menu on outside click
const closeImageReplaceMenu = () => {
  showImageReplaceMenu.value = false;
};

onMounted(() => {
  document.addEventListener('click', closeImageReplaceMenu);
});

onUnmounted(() => {
  document.removeEventListener('click', closeImageReplaceMenu);
});

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
      const textColor = isDarkMode.value ? '#f1f5f9' : '#1e293b';
      const bgColor = isDarkMode.value ? '#1e293b' : 'white';
      
      // Update textarea caret color
      htmlOutputRef.value.style.caretColor = textColor;
      
      // Update highlighter colors
      if (highlighterRef.value) {
        highlighterRef.value.style.backgroundColor = bgColor;
        highlighterRef.value.style.color = textColor;
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

// Handle right-click on image
const handleImageRightClick = (event) => {
  if (event.target.tagName === 'IMG') {
    event.preventDefault();
    targetImage.value = event.target;
    
    const rect = editorRef.value.getBoundingClientRect();
    imageReplaceMenuStyle.value = {
      left: `${event.clientX - rect.left}px`,
      top: `${event.clientY - rect.top}px`,
    };
    showImageReplaceMenu.value = true;
  }
};

// Replace image
const replaceImage = () => {
  if (targetImage.value && newImageUrl.value) {
    let imgSrc;
    const googleDriveId = extractGoogleDriveImageId(newImageUrl.value);

    if (googleDriveId) {
      imgSrc = `https://drive.google.com/thumbnail?id=${googleDriveId}&sz=s4000`;
    } else if (isValidUrl(newImageUrl.value)) {
      imgSrc = newImageUrl.value;
    } else {
      alert('Invalid image URL. Please enter a valid URL or Google Drive link.');
      return;
    }

    targetImage.value.src = imgSrc;
    handleEditorInput();
    newImageUrl.value = '';
    showImageReplaceMenu.value = false;
  }
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

// ---- Highlighting and Sync Scroll Logic ----

const escapeHtml = (str) => {
  if (typeof str !== 'string') return '';
  return str.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;");
};

/**
 * Finds the starting position of the Nth occurrence of a substring.
 * @param {string} string The string to search in.
 * @param {string} substring The substring to search for.
 * @param {number} n The zero-based index of the occurrence to find.
 * @returns {number} The starting index, or -1 if not found.
 */
const findNthOccurrence = (string, substring, n) => {
  let index = -1;
  for (let i = 0; i <= n; i++) {
    index = string.indexOf(substring, index + 1);
    if (index === -1) {
      return -1; // Nth occurrence not found
    }
  }
  return index;
};

const updateHighlighterContent = (content, highlightInfo = null) => {
  if (!highlighterRef.value) return;

  if (!highlightInfo) {
    highlighterRef.value.innerHTML = escapeHtml(content);
    return;
  }

  let startPos = -1;
  let endPos = -1;

  // NEW: Logic for IMG tags using their overall index in the document
  if (highlightInfo.tag === 'IMG') {
    const { index } = highlightInfo;
    let searchFrom = -1;
    // Find the starting position of the nth '<img' tag
    for (let i = 0; i <= index; i++) {
      // Case-insensitive search
      searchFrom = content.toLowerCase().indexOf('<img', searchFrom + 1);
      if (searchFrom === -1) {
        break; // Nth image tag not found in the string
      }
    }

    if (searchFrom !== -1) {
      startPos = searchFrom;
      // Find the closing '>' of that specific tag
      const tagEnd = content.indexOf('>', startPos);
      if (tagEnd !== -1) {
        endPos = tagEnd + 1; // Include the '>' in the highlight
      }
    }
  } else { // ORIGINAL LOGIC for all other tags based on outerHTML
    const { elementHTML, index } = highlightInfo;
    startPos = findNthOccurrence(content, elementHTML, index);
    if (startPos !== -1) {
      endPos = startPos + elementHTML.length;
    }
  }

  // Common rendering logic if a valid position was found
  if (startPos !== -1 && endPos !== -1) {
    const before = content.substring(0, startPos);
    const highlight = content.substring(startPos, endPos);
    const after = content.substring(endPos);

    // Escape each part individually and reconstruct with the <mark> tag.
    const finalContent =
      escapeHtml(before) +
      `<mark class="highlight" id="highlight-target">${escapeHtml(highlight)}</mark>` +
      escapeHtml(after);

    highlighterRef.value.innerHTML = finalContent;
  } else {
    // Fallback if no match was found (e.g., due to DOM/string desync)
    highlighterRef.value.innerHTML = escapeHtml(content);
  }
};


watch(currentHtml, (newHtml) => {
  updateHighlighterContent(newHtml);
  if (htmlOutputRef.value && highlighterRef.value) {
      highlighterRef.value.scrollTop = htmlOutputRef.value.scrollTop;
      highlighterRef.value.scrollLeft = htmlOutputRef.value.scrollLeft;
  }
});

const handleVisualEditorScroll = () => {
  isVisualEditorScrolling.value = true;
  clearTimeout(visualScrollTimeout);
  visualScrollTimeout = setTimeout(() => {
    isVisualEditorScrolling.value = false;
  }, 150);
};

const handleVisualEditorHover = (event) => {
  if (isVisualEditorScrolling.value) return;

  clearTimeout(highlightDebounceTimer);
  highlightDebounceTimer = setTimeout(() => {
    let target = event.target;
    if (target.nodeType === 3) { // If it's a text node, get its parent element.
      target = target.parentElement;
    }

    if (!target || !editorRef.value.contains(target) || target === editorRef.value) {
      clearHighlight();
      return;
    }

    const tagName = target.tagName;

    if (!tagName) {
      clearHighlight();
      return;
    }

    try {
      let highlightInfo = null;

      // NEW: Special case for image tags to use nth-of-type logic
      if (tagName === 'IMG') {
        // Find all rendered image elements
        const allImages = Array.from(editorRef.value.querySelectorAll('img'));
        // Find the specific index of the hovered image
        const imageIndex = allImages.indexOf(target);

        if (imageIndex > -1) {
          // Create highlight info based on the tag and its index
          highlightInfo = {
            tag: 'IMG',
            index: imageIndex
          };
        }
      } else { // ORIGINAL: Logic for all other tags
        const targetOuterHTML = target.outerHTML;
        // Find all elements of the same tag type in the editor
        const allElementsOfSameType = editorRef.value.querySelectorAll(tagName);
        // Filter to get only elements with the exact same outerHTML as our target
        const identicalElements = Array.from(allElementsOfSameType).filter(el => el.outerHTML === targetOuterHTML);
        // Find the index of our specific hovered element among its identical peers
        const hoverIndex = identicalElements.indexOf(target);

        if (hoverIndex > -1) {
          // Pass the unique element HTML and its specific index to the highlighter
          highlightInfo = {
            elementHTML: targetOuterHTML,
            index: hoverIndex
          };
        }
      }

      if (highlightInfo) {
        updateHighlighterContent(currentHtml.value, highlightInfo);

        nextTick(() => {
          const highlightEl = highlighterRef.value.querySelector('#highlight-target');
          if (highlightEl && highlighterRef.value && htmlOutputRef.value) {
            const codePanel = htmlOutputRef.value;
            const scrollDistance = Math.abs(highlightEl.offsetTop - codePanel.scrollTop);
            const scrollThreshold = codePanel.clientHeight * 2;
            const behavior = scrollDistance > scrollThreshold ? 'instant' : 'smooth';

            highlightEl.scrollIntoView({
              behavior,
              block: 'center',
              inline: 'nearest'
            });

            const syncScrollAction = () => {
              if (htmlOutputRef.value) {
                htmlOutputRef.value.scrollTop = highlighterRef.value.scrollTop;
                htmlOutputRef.value.scrollLeft = highlighterRef.value.scrollLeft;
              }
            };

            if (behavior === 'instant') {
              syncScrollAction();
            } else {
              setTimeout(syncScrollAction, 400);
            }
          }
        });
      } else {
        clearHighlight();
      }
    } catch (error) {
      console.error('Error during hover handling:', error);
      clearHighlight();
    }
  }, 50);
};


const clearHighlight = () => {
  clearTimeout(highlightDebounceTimer);
  updateHighlighterContent(currentHtml.value);
};

// Syncs scroll from user scrolling the textarea
const syncScroll = (event) => {
  if (highlighterRef.value) {
    highlighterRef.value.scrollTop = event.target.scrollTop;
    highlighterRef.value.scrollLeft = event.target.scrollLeft;
  }
};

// --- UPDATED Resizing Logic ---
const startResize = () => {
  isDragging.value = true;
  document.body.style.userSelect = 'none'; // Prevent text selection during drag
  document.addEventListener('mousemove', resize);
  document.addEventListener('mouseup', stopResize);
};

const resize = (event) => {
  if (!isDragging.value) return;

  const mainContent = mainContentRef.value;
  if (!mainContent) return;

  // UPDATED: Check for mobile view to switch between horizontal and vertical resizing
  if (isMobileView.value) {
    // Vertical resizing logic
    const { top, height } = mainContent.getBoundingClientRect();
    const newEditorHeight = event.clientY - top;
    const newCodeHeight = height - newEditorHeight;

    // Set minimum height for panels
    if (newEditorHeight > 100 && newCodeHeight > 100) {
      editorHeight.value = `${newEditorHeight}px`;
      codeHeight.value = `${newCodeHeight}px`;
    }
  } else {
    // Horizontal resizing logic (original)
    const { left, width } = mainContent.getBoundingClientRect();
    const newEditorWidth = event.clientX - left;
    const newCodeWidth = width - newEditorWidth;

    // Set minimum width for panels
    if (newEditorWidth > 100 && newCodeWidth > 100) {
      editorWidth.value = `${newEditorWidth}px`;
      codeWidth.value = `${newCodeWidth}px`;
    }
  }
};

const stopResize = () => {
  isDragging.value = false;
  document.body.style.userSelect = ''; // Re-enable text selection
  document.removeEventListener('mousemove', resize);
  document.removeEventListener('mouseup', stopResize);
};

// NEW: Function to check screen size and set mobile view state
const checkScreenSize = () => {
  isMobileView.value = window.innerWidth <= 768;
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

  // Setup visual editor scroll listener
  if (editorRef.value && editorRef.value.parentElement) {
    editorRef.value.parentElement.addEventListener('scroll', handleVisualEditorScroll);
  }

  // Initial highlighter content
  updateHighlighterContent(currentHtml.value);

  // NEW: Add window resize listener to handle layout changes
  window.addEventListener('resize', checkScreenSize);
  checkScreenSize(); // Initial check on mount
});

// NEW: Clean up the event listener on component unmount
onUnmounted(() => {
  window.removeEventListener('resize', checkScreenSize);
});
</script>

<style scoped>
/* --- NEW: Keyframe animations --- */
@keyframes jiggle {
  0%, 100% { transform: rotate(-1deg); }
  50% { transform: rotate(1deg); }
}

@keyframes fadeInScale {
  from {
    opacity: 0;
    transform: scale(0.95);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

@keyframes popIn {
  from {
    opacity: 0;
    transform: scale(0.8);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

/* Base styles and variables */
.html-editor {
  --primary-color: #2d3748;
  --primary-hover: #1a202c;
  --primary-focus-glow: rgba(45, 55, 72, 0.4);
  --secondary-color: #4a5568;
  --secondary-hover: #2d3748;
  --danger-color: #e53e3e;
  --danger-hover: #c53030;
  --text-color: #1e293b;
  --text-secondary: #475569;
  --bg-color: #ffffff;
  --bg-secondary: #f8fafc;
  --border-color: #e2e8f0;
  --shadow-color: rgba(45, 55, 72, 0.1);
  --shadow-hover-color: rgba(45, 55, 72, 0.2);
  --scrollbar-bg: #f1f5f9;
  --scrollbar-thumb: #94a3b8;
  --bracket-color: var(--text-color);
  --h-bg-color: #000000;
  --h-text-color: #ffffff;
  --highlight-bg: #fde68a; /* Dull amber yellow */
  --ruled-line-color: #d1d5db;

  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
  
  display: flex;
  flex-direction: column;
  
  color: var(--text-color);
  background-color: var(--bg-color);
  transition: background-color 0.4s ease, color 0.4s ease;
  
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
  --primary-focus-glow: rgba(148, 163, 184, 0.4);
  --secondary-color: #4a5568;
  --secondary-hover: #2d3748;
  --danger-color: #f87171;
  --danger-hover: #ef4444;
  --text-color: #f1f5f9;
  --text-secondary: #cbd5e1;
  --bg-color: #0f172a;
  --bg-secondary: #1e293b;
  --border-color: #334155;
  --shadow-color: rgba(0, 0, 0, 0.2);
  --shadow-hover-color: rgba(0, 0, 0, 0.4);
  --scrollbar-bg: #1e293b;
  --scrollbar-thumb: #64748b;
  --bracket-color: var(--text-color);
  --h-bg-color: #ffffff;
  --h-text-color: #000000;
  --highlight-bg: #fBBF2480;
  --ruled-line-color: #475569;
}

/* App header */
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
  transition: border-color 0.4s ease;
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

/* Styles for Logo Animation */
.logo-letters {
  display: flex;
  position: relative;
}

.bracket-button {
  background: none;
  border: none;
  padding: 0 5px;
  font-family: inherit;
  font-size: inherit;
  font-weight: 500;
  color: var(--bracket-color);
  cursor: pointer;
  border-radius: 4px;
  transition: opacity 0.3s ease, transform 0.3s ease;
}
.bracket-button:hover {
  opacity: 0.7;
  transform: scale(1.1);
}

/* --- Vue TransitionGroup styles for the logo --- */
.logo-move,
.logo-enter-active,
.logo-leave-active {
  transition: all 0.5s cubic-bezier(0.55, 0, 0.1, 1);
}
.logo-enter-from,
.logo-leave-to {
  opacity: 0;
  transform: scaleY(0.01) translate(30px, 0);
}
.logo-leave-active {
  position: absolute;
}

.letter, .letter-h {
  margin: 0 1px;
  transition: all 0.4s ease;
}

.letter-h {
  color: var(--h-text-color);
  background-color: var(--h-bg-color);
  padding: 0 5px;
  border-radius: 4px;
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
  transition: all 0.3s ease;
  flex-shrink: 0;
  margin-left: auto;
}

.theme-toggle:hover {
  background-color: var(--bg-secondary);
  transform: scale(1.1) rotate(15deg);
  animation: jiggle 0.4s infinite;
}

/* Toolbar */
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
  transition: background-color 0.4s ease, box-shadow 0.4s ease;
}

.toolbar-section {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  align-items: center;
}

/* Buttons */
.btn {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 12px;
  border-radius: 6px;
  font-weight: 500;
  font-size: 0.85rem;
  cursor: pointer;
  border: none;
  white-space: nowrap;
  transition: all 0.2s cubic-bezier(0.25, 0.8, 0.25, 1);
  box-shadow: 0 1px 3px var(--shadow-color);
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

.btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px var(--shadow-hover-color);
  animation: jiggle 0.4s;
}

.btn:active:not(:disabled) {
  transform: translateY(0px);
  box-shadow: 0 1px 3px var(--shadow-color);
}

.btn-primary {
  background-color: var(--primary-color);
  color: white;
}
.btn-primary:hover:not(:disabled) {
  background-color: var(--primary-hover);
}

.btn-secondary {
  background-color: var(--secondary-color);
  color: white;
}
.btn-secondary:hover:not(:disabled) {
  background-color: var(--secondary-hover);
}

.btn-danger {
  background-color: var(--danger-color);
  color: white;
}
.btn-danger:hover:not(:disabled) {
  background-color: var(--danger-hover);
}

.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.hidden {
  display: none;
}

/* Main content layout */
.main-content {
  display: flex;
  padding: 15px;
  flex-grow: 1;
  overflow: hidden;
  box-sizing: border-box;
}

.panel {
  background-color: var(--bg-secondary);
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 8px var(--shadow-color);
  display: flex;
  flex-direction: column;
  transition: background-color 0.4s ease, box-shadow 0.4s ease;
}

@media (max-width: 768px) {
  .main-content {
    flex-direction: column;
    padding: 10px;
  }

  .panel {
    width: 100%;
    height: 50%;
  }
}

.panel-header {
  background-color: var(--primary-color);
  color: white;
  padding: 10px 15px;
  flex-shrink: 0;
  transition: background-color 0.4s ease;
}

.panel-title {
  margin: 0;
  font-size: 1.1rem;
  font-weight: 600;
  color: white;
}

.editor-container {
  border: 1px solid var(--border-color);
  border-radius: 0 0 8px 8px;
  padding: 15px;
  position: relative;
  flex-grow: 1;
  overflow: auto;
  background-color: white;
  transition: border-color 0.4s ease;
}

/* Custom scrollbar styles */
.editor-container::-webkit-scrollbar,
.html-output-textarea::-webkit-scrollbar {
  width: 10px;
  height: 10px;
}

.editor-container::-webkit-scrollbar-track,
.html-output-textarea::-webkit-scrollbar-track {
  background: var(--scrollbar-bg);
}

.editor-container::-webkit-scrollbar-thumb,
.html-output-textarea::-webkit-scrollbar-thumb {
  background-color: var(--scrollbar-thumb);
  border-radius: 5px;
  border: 2px solid var(--scrollbar-bg);
}

.editor-content {
  outline: none;
  min-height: 100%;
  color: #1e293b;
  background-color: white;
}

.editor-content img, .editor-content .editor-image {
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

.code-editor-container {
  position: relative;
  flex-grow: 1;
  border-radius: 0 0 8px 8px;
  overflow: hidden;
}

.code-highlighter,
.html-output-textarea {
  font-family: 'Fira Code', monospace;
  font-size: 14px;
  line-height: 1.5;
  padding: 15px;
  margin: 0;
  white-space: pre-wrap;
  word-break: break-all;
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  box-sizing: border-box;
  border: none;
  outline: none;
  transition: color 0.4s ease, background-color 0.4s ease;
}

.html-output-textarea {
  background-color: transparent;
  color: transparent;
  caret-color: var(--text-color);
  resize: none;
  overflow: auto;
  z-index: 2;
}

.code-highlighter {
  pointer-events: none;
  z-index: 1;
  overflow: hidden;
  text-align: left;
  background-image: repeating-linear-gradient(
    transparent 0,
    transparent calc(1.5em - 1px),
    var(--ruled-line-color) calc(1.5em - 1px),
    var(--ruled-line-color) 1.5em
  );
  background-size: 100% 1.5em;
  background-attachment: local;
  background-position-y: 15px;
}

.code-highlighter :deep(.highlight) {
  background-color: var(--highlight-bg);
  border-radius: 3px;
}

/* Bubbles */
.image-bubble, .link-bubble {
  position: absolute;
  z-index: 10;
  animation: popIn 0.3s ease-out forwards;
}

.image-replace-menu {
  position: absolute;
  z-index: 10;
  background-color: var(--bg-color);
  border: 1px solid var(--border-color);
  border-radius: 6px;
  padding: 8px;
  box-shadow: 0 4px 12px var(--shadow-hover-color);
  display: flex;
  gap: 8px;
  animation: popIn 0.2s ease-out forwards;
}

.image-replace-menu input {
  border: 1px solid var(--border-color);
  border-radius: 4px;
  padding: 6px;
  font-size: 0.9rem;
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
  transition: all 0.2s ease;
}
.bubble-btn:hover {
  background-color: var(--primary-hover);
  transform: scale(1.1);
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
  box-shadow: 0 4px 20px var(--shadow-hover-color);
  animation: fadeInScale 0.3s ease-out forwards;
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
  transition: all 0.3s ease;
}

.modal-input:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 3px var(--primary-focus-glow);
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

.resizer {
  width: 5px;
  cursor: ew-resize;
  background-color: var(--border-color);
  flex-shrink: 0;
  transition: background-color 0.3s ease;
}
.resizer:hover {
  background-color: var(--primary-color);
}

@media (max-width: 768px) {
  .resizer {
    width: 100%;
    height: 4px;
    cursor: ns-resize;
    margin: 3px 0;
    border-radius: 2px;
  }
}
</style>