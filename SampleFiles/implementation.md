# Status
SUCCESS:Implementation guide generated for the Rules Editor (BETA) context menu and keyboard shortcuts feature.
Confidence: 74%

# Implementation Guide
Issue: IDEA-30 — Context menu and keyboard shortcuts for Rules Editor components
Date: 2026-11-24
Objective: Add a right-click context menu, mirrored keyboard shortcuts, and a delete confirmation to Action Taker and Info Collector nodes in the Rules Editor (BETA, `RulesEditor_v2`), so a user can enable/disable, reorder, move an Action Taker to a neighbouring Decision Maker, or delete a component (with confirmation) without opening its edit dialog or redrawing connections, exactly as specified in `design.md`'s Proposed Solution and Impacted Components.

# Preconditions
- Environment: Node.js toolchain for `RulesEditor_v2/` (React 18.2, ReactFlow 11.7.4, MUI 5.14, `react-scripts` 5.0.1 / CRA Jest runner), runnable with `npm install` then `npm start`/`npm test`/`npm run build` from `RulesEditor_v2/`. No new dependency is added; `@mui/material`'s `Menu`/`MenuItem` and `@mui/icons-material`'s `ToggleOn`/`ToggleOff`/`ArrowDownward`/`ArrowUpward`/`ArrowForward`/`ArrowBack`/`Edit`/`DeleteForever` are already in `package.json`.
- Branch/version: work against the current `RulesEditor_v2/src` tree as read (no other in-flight change to `RuleEditor.js`, `RuleNode.jsx`, `DialogBox.jsx`, or `ReactFlowUtils.js` assumed).
- Repository-level instructions and skills that apply, and what each requires here:
  - `AGENTS.md` (repo-wide index): routes any `RulesEditor_v2/**` change to `.github/instructions/rulesengine.instructions.md` and the `rulesengine-react-editor` skill; both are applied below.
  - `.github/instructions/rulesengine.instructions.md` (`applyTo: java/**/rulesengine/**,webapps/admin/re_*,webapps/admin/rulesTranslator.jsp,RulesEditor/**,RulesEditor_v2/**`): requires (a) rebuilding and committing `webapps/RuleEditor_v2.zip` in the same change — "a change to editor source is not finished until its zip ... is rebuilt and committed"; (b) any new HTTP call goes in `src/utils/Api.js` with its existing double-encoding helpers — not applicable here, no new server call is added; (c) MUI only for every control, `.jsx` for new React components and `.js` for new plain helpers.
  - `.github/skills/rulesengine-react-editor/SKILL.md`: for "Canvas behaviour" changes, the file map directs to `src/RuleEditor.js` and `src/utils/ReactFlowUtils.js` (matches design.md's Impacted Components); requires testing with `npm start` against the canned XML in `public/data/`, building with `deploy.bat` (`npm run build` + zip), and opening `/odsadmin/rule_editor_v2/index.html` to confirm end-to-end behaviour; requires updating `docs/rulesengine/editors-and-ui.md` after any change that affects how the editor works.
  - No path-scoped `.github/instructions/*.instructions.md` files other than the above match `RulesEditor_v2/**`; no reporting/VEE/compassAPI instructions apply.
- Every new or changed file below is `.jsx` (React component) or `.js` (plain helper), matching the skill's naming rule.
- No database, Java, or `rulesTranslator.jsp` change is needed (design.md: "Other Systems: none").

# Step-by-Step Code Changes

1. File: `RulesEditor_v2/src/utils/ReactFlowUtils.js`
   Type: Modify
   Intent: Add the pure, stateless graph helpers that (a) compute each new menu item's availability for a given Action Taker or Info Collector node, and (b) compute the updated edge lists for Move Forward/Backward and Send to Next/Previous Decision Maker, so both `ComponentContextMenu.jsx`/`RuleNode.jsx` (availability) and `RuleEditor.js` (mutation) reuse one implementation instead of duplicating graph-walking logic, per design.md's Impacted Components ("add helper(s) to compute each menu item's availability ... alongside the existing `hasCycles`/`getEdgeParams` graph helpers").
   Why: Realizes design.md's Change Plan step 1 and Proposed Solution's "Order/parentage mutation" bullet. Putting the edge-list computation itself here (not only the availability booleans) is a narrowing choice: `ReactFlowUtils.js` is already this codebase's home for pure, non-React graph functions (`hasCycles`, `hasDisconnectedNodes`, `repositionNodes`, `compareNodeArrays`/`compareEdgeArrays`), and design.md's Validation Strategy asks for unit tests of this logic "following the precedent of `src/utils/Marshaller.test.js`" — a plain-Jest test of an exported pure function. `RuleEditor.js`'s `useCallback` hooks are not unit-testable in isolation the same way, so the edge-computation logic is placed here and the `RuleEditor.js` callbacks (Step 2) become thin wrappers that call these functions and then `setNodes`/`setEdges`. `RuleEditor.js` is still the file that *performs* the mutation (calls `setNodes`/`setEdges`), matching design.md's Impacted Components line for `RuleEditor.js`.
   Suggested Change:
   ```js
   // Added near the other pure graph helpers (hasCycles, repositionNodes), reusing the
   // same edge-handle conventions already used throughout RuleEditor.js:
   // - a Decision-Maker-to-its-first-Action-Taker edge has sourceHandle 'DM_to_AT_<dmId>'
   // - a Decision-Maker-to-Decision-Maker edge has targetHandle 'DM_to_DM_<targetDmId>'
   // - every other Action Taker/Info Collector chain edge is a plain source/target edge
   import { decisionMakerId, actionTakerId } from './Utils.js';

   // Returns the Decision Maker reached by the current Decision Maker's single outgoing
   // DM-to-DM edge, or undefined if there is none or more than one (A-1).
   export const getNextDecisionMaker = (decisionMaker, nodes, edges) => {
     const outgoing = edges.filter(e => e.source === decisionMaker.id && e.targetHandle && e.targetHandle.includes('DM_to_DM'));
     if (outgoing.length !== 1) return undefined;
     return nodes.find(n => n.id === outgoing[0].target);
   };

   // Returns the Decision Maker whose single outgoing DM-to-DM edge reaches this Decision
   // Maker, or undefined if there is none or more than one (A-1).
   export const getPreviousDecisionMaker = (decisionMaker, nodes, edges) => {
     const incoming = edges.filter(e => e.target === decisionMaker.id && e.targetHandle && e.targetHandle.includes('DM_to_DM'));
     if (incoming.length !== 1) return undefined;
     return nodes.find(n => n.id === incoming[0].source);
   };

   // Finds the edge immediately before (incoming to) and immediately after (outgoing from)
   // a node in its own component-type chain, mirroring onNodesDelete's/onNodeDragStop's
   // existing prevNodeId/nextNodeId lookup.
   const findChainNeighborEdges = (nodeId, edges) => ({
     incoming: edges.find(e => e.target === nodeId),
     outgoing: edges.find(e => e.source === nodeId && (!e.sourceHandle || !e.sourceHandle.includes('DM_to_AT_'))),
   });

   // Action Taker menu availability: Move Forward/Backward within the same Decision Maker,
   // Send to Next/Previous Decision Maker across the rule (REQ-2.4, A-1, A-7).
   export const getActionTakerMenuAvailability = (node, nodes, edges) => {
     const { incoming, outgoing } = findChainNeighborEdges(node.id, edges);
     const parentDecisionMaker = nodes.find(n => n.id === node.parentNode);
     return {
       canMoveForward: !!outgoing,
       canMoveBackward: !!(incoming && incoming.source !== node.parentNode),
       canSendToNext: !!(parentDecisionMaker && getNextDecisionMaker(parentDecisionMaker, nodes, edges)),
       canSendToPrevious: !!(parentDecisionMaker && getPreviousDecisionMaker(parentDecisionMaker, nodes, edges)),
     };
   };

   // Info Collector menu availability: Move Forward/Backward within the rule-wide
   // Info Collector sequence (REQ-3.4, A-7).
   export const getInfoCollectorMenuAvailability = (node, nodes, edges) => {
     const { incoming, outgoing } = findChainNeighborEdges(node.id, edges);
     const nextIsInfoCollector = outgoing && nodes.find(n => n.id === outgoing.target)?.type === 'infoCollector';
     const prevIsInfoCollector = incoming && nodes.find(n => n.id === incoming.source)?.type === 'infoCollector';
     return {
       canMoveForward: !!nextIsInfoCollector,
       canMoveBackward: !!prevIsInfoCollector,
     };
   };

   // Computes the updated edge array that swaps `nodeId` with its chain successor
   // `neighborId` (REQ-2.6/REQ-2.7/REQ-3.6/REQ-3.7's worked examples: A,B,C,D,E -> A,B,D,C,E).
   // Only the edges touching P, the two swapped nodes, and (if present) N2 change.
   export const swapChainNeighbors = (nodeId, neighborId, nodes, edges, getEdgeId) => {
     const { incoming: pToSelected } = findChainNeighborEdges(nodeId, edges);
     const { outgoing: neighborToN2 } = findChainNeighborEdges(neighborId, edges);
     const pId = pToSelected ? pToSelected.source : undefined;
     const n2Id = neighborToN2 ? neighborToN2.target : undefined;

     let updated = edges.filter(e =>
       !((e.source === nodeId || e.target === nodeId) || (e.source === neighborId || e.target === neighborId))
     );

     if (pId !== undefined) {
       // Preserve the DM_to_AT_ handle/type only when P is the Decision Maker parent.
       const pIsParentDM = pId === nodes.find(n => n.id === nodeId).parentNode;
       updated = updated.concat(pIsParentDM
         ? { id: getEdgeId(), source: pId, sourceHandle: 'DM_to_AT_' + pId, target: neighborId, zIndex: 1001, type: 'default' }
         : { id: getEdgeId(), source: pId, target: neighborId, zIndex: 1001, type: 'floating' });
     }
     updated = updated.concat({ id: getEdgeId(), source: neighborId, target: nodeId, zIndex: 1001, type: 'floating' });
     if (n2Id !== undefined) {
       updated = updated.concat({ id: getEdgeId(), source: nodeId, target: n2Id, zIndex: 1001, type: 'floating' });
     }
     return updated;
   };
   ```

2. File: `RulesEditor_v2/src/RuleEditor.js`
   Type: Modify
   Intent: Add the graph-mutating callbacks `moveActionTakerForward`, `moveActionTakerBackward`, `moveInfoCollectorForward`, `moveInfoCollectorBackward`, `sendActionTakerToDecisionMaker(id, direction)`, and `requestDeleteComponent(id, type)`; add the document-level keydown listener for the per-action shortcuts (REQ-4.9); route Delete through the existing `setConfirmBox`/`confirmMsg`/`closeConfirmBox` state (already defined at lines 71, 92–98) and the existing `ConfirmationBox` instance (already rendered at line 1287); prop-drill the new callbacks into every node's `data`, alongside the existing `saveNodeData`.
   Why: Realizes design.md's Impacted Components line for `RuleEditor.js` and Change Plan steps 2, 4, 5. Reuses `useReactFlow().deleteElements` so the existing `onNodesDelete` reconnection/resize logic (lines 1205–1253) is untouched, and reuses `onNodeDragStop`'s reparenting fields (`parentNode`/`extent`/`expandParent`) for Send to Next/Previous Decision Maker, updating the existing node in place instead of building a drag-stop replica node (Proposed Solution, Alternatives Considered).
   Suggested Change:
   ```js
   // New imports alongside the existing ReactFlowUtils.js import (line 4):
   import {
     hasDisconnectedNodes, hasCycles, repositionNodes,
     getNextDecisionMaker, getPreviousDecisionMaker, swapChainNeighbors,
   } from './utils/ReactFlowUtils.js';
   import { useReactFlow } from 'reactflow'; // alongside the existing reactflow import
   import { isMacPlatform, isEditableFocus } from './utils/Utils.js';

   // Inside RuleEditor, alongside reactFlowInstance/setReactFlowInstance (line 61):
   const { deleteElements } = useReactFlow();

   // Toggling enabled reuses the existing saveNodeData path (no new mutation style),
   // matching REQ-2.5/REQ-3.5's "no confirmation, no notification" (Section 5.5).
   const toggleEnabled = useCallback((nodeId) => {
     const node = reactFlowInstance.getNodes().find(n => n.id === nodeId);
     const isEnabled = node.data.nodeData.enabled === undefined ? true : Utils.isTrue(node.data.nodeData.enabled);
     saveNodeData(node.data.label, nodeId, node.data.description, node.data.nodeData.parameters, !isEnabled);
   }, [reactFlowInstance, saveNodeData]);

   // Move Forward/Backward for Action Takers within the same Decision Maker (REQ-2.6/REQ-2.7).
   // Backward is the mirror of Forward: swapping X with its predecessor P is the same
   // operation as swapping P with its successor X, so Backward delegates to the same helper.
   const moveActionTakerForward = useCallback((nodeId) => {
     const currentNodes = reactFlowInstance.getNodes();
     const currentEdges = reactFlowInstance.getEdges();
     const { outgoing } = /* ReactFlowUtils.findChainNeighborEdges is internal; call getActionTakerMenuAvailability first to guard */ { outgoing: currentEdges.find(e => e.source === nodeId) };
     if (!outgoing) return; // already last (A-7): no effect
     setEdges(swapChainNeighbors(nodeId, outgoing.target, currentNodes, currentEdges, getEdgeId));
   }, [reactFlowInstance, setEdges]);

   const moveActionTakerBackward = useCallback((nodeId) => {
     const currentEdges = reactFlowInstance.getEdges();
     const incoming = currentEdges.find(e => e.target === nodeId);
     const node = reactFlowInstance.getNodes().find(n => n.id === nodeId);
     if (!incoming || incoming.source === node.parentNode) return; // already first (A-7): no effect
     moveActionTakerForward(incoming.source);
   }, [reactFlowInstance, moveActionTakerForward]);

   // Move Forward/Backward for Info Collectors in the rule-wide sequence (REQ-3.6/REQ-3.7);
   // same swap helper, gated by getInfoCollectorMenuAvailability's canMoveForward/Backward.
   const moveInfoCollectorForward = useCallback((nodeId) => {
     const currentNodes = reactFlowInstance.getNodes();
     const currentEdges = reactFlowInstance.getEdges();
     const outgoing = currentEdges.find(e => e.source === nodeId);
     if (!outgoing || currentNodes.find(n => n.id === outgoing.target)?.type !== 'infoCollector') return;
     setEdges(swapChainNeighbors(nodeId, outgoing.target, currentNodes, currentEdges, getEdgeId));
   }, [reactFlowInstance, setEdges]);

   const moveInfoCollectorBackward = useCallback((nodeId) => {
     const currentNodes = reactFlowInstance.getNodes();
     const currentEdges = reactFlowInstance.getEdges();
     const incoming = currentEdges.find(e => e.target === nodeId);
     if (!incoming || currentNodes.find(n => n.id === incoming.source)?.type !== 'infoCollector') return;
     moveInfoCollectorForward(incoming.source);
   }, [reactFlowInstance, moveInfoCollectorForward]);

   // Send to Next/Previous Decision Maker (REQ-2.8/REQ-2.9): reparent the existing Action
   // Taker node in place (no replica node, unlike onNodeDragStop), modeled on the same
   // "bridge the gap, then append after the target's last Action Taker" logic as
   // onNodeDragStop (lines 1102-1191) and onNodesDelete (lines 1205-1253).
   const sendActionTakerToDecisionMaker = useCallback((nodeId, direction) => {
     const currentNodes = reactFlowInstance.getNodes();
     const currentEdges = reactFlowInstance.getEdges();
     const node = currentNodes.find(n => n.id === nodeId);
     const currentDM = currentNodes.find(n => n.id === node.parentNode);
     const targetDM = direction === 'next'
       ? getNextDecisionMaker(currentDM, currentNodes, currentEdges)
       : getPreviousDecisionMaker(currentDM, currentNodes, currentEdges);
     if (!targetDM) return; // no such neighbour (A-1): no effect

     // Bridge the gap left in the current Decision Maker (same reconnect pattern as
     // onNodesDelete lines 1211-1222).
     const incomingEdge = currentEdges.find(e => e.target === nodeId);
     const outgoingEdge = currentEdges.find(e => e.source === nodeId && (!e.sourceHandle || !e.sourceHandle.includes('DM_to_AT_')));
     let updatedEdges = currentEdges.filter(e => !(e.source === nodeId || e.target === nodeId));
     if (incomingEdge && outgoingEdge) {
       const prevNodeId = incomingEdge.source;
       const nextNodeId = outgoingEdge.target;
       updatedEdges = updatedEdges.concat(prevNodeId === node.parentNode
         ? { id: getEdgeId(), source: prevNodeId, sourceHandle: 'DM_to_AT_' + prevNodeId, target: nextNodeId, zIndex: 1001, type: 'default' }
         : { id: getEdgeId(), source: prevNodeId, target: nextNodeId, zIndex: 1001, type: 'floating' });
     }

     // Append after the target Decision Maker's last Action Taker, or become its first
     // (same two-branch wiring as onNodeDragStop lines 1170-1182).
     const targetChildren = currentNodes.filter(n => n.parentNode === targetDM.id);
     updatedEdges = updatedEdges.concat(targetChildren.length > 0
       ? { id: getEdgeId(), source: targetChildren[targetChildren.length - 1].id, target: nodeId, zIndex: 1001 }
       : { id: getEdgeId(), source: targetDM.id, sourceHandle: 'DM_to_AT_' + targetDM.id, target: nodeId, zIndex: 1001, type: 'default' });

     setEdges(updatedEdges);
     setNodes(currentNodes.map(n => n.id === nodeId
       ? { ...n, parentNode: targetDM.id, data: { ...n.data, parentNode: targetDM.id }, extent: 'parent', expandParent: true,
           position: { x: xOffset, y: yOffset + (targetChildren.length * yGap) } }
       : n
     ));
     // Per REQ-2.6/REQ-3.6 and design.md's "Repositioning after Move/Send": no further
     // repositioning is done here; the user runs the existing Auto Arrange action.
   }, [reactFlowInstance, setNodes, setEdges]);

   // Delete confirmation (REQ-6.2-REQ-6.8): stop the bare Delete/Backspace path, open the
   // reused ConfirmationBox with per-type wording (A-12), and only remove the node via
   // deleteElements on OK, so onNodesDelete's existing reconnection/resize logic still runs.
   const requestDeleteComponent = useCallback((nodeId, nodeType) => {
     const label = nodeType === 'actionTaker' ? 'Action Taker' : 'Info Collector';
     setConfirmBox(`Are you sure you want to delete this ${label}?`, () => deleteElements({ nodes: [{ id: nodeId }] }));
   }, [deleteElements]);

   // Document-level listener so a per-action shortcut (Ctrl+Alt+<letter> / Cmd+Option+<letter>)
   // acts on the single selected Action Taker/Info Collector "regardless of whether its
   // context menu is currently open" (REQ-4.9), even when the node itself lacks DOM focus.
   useEffect(() => {
     const shortcutMap = { t: 'toggle', j: 'forward', k: 'backward', l: 'next', h: 'previous', o: 'edit', d: 'delete' };
     const handleDocumentKeyDown = (e) => {
       const modifierPressed = isMacPlatform() ? (e.metaKey && e.altKey) : (e.ctrlKey && e.altKey);
       if (!modifierPressed || isEditableFocus(document.activeElement)) return; // REQ-4.10/A-3
       const action = shortcutMap[e.key.toLowerCase()];
       if (!action) return;

       const selected = reactFlowInstance.getNodes().filter(n => n.selected && (n.type === 'actionTaker' || n.type === 'infoCollector'));
       if (selected.length !== 1) return; // REQ-4.9: acts on exactly one selected component
       const node = selected[0];
       e.preventDefault();
       if (action === 'toggle') toggleEnabled(node.id);
       else if (action === 'forward') node.type === 'actionTaker' ? moveActionTakerForward(node.id) : moveInfoCollectorForward(node.id);
       else if (action === 'backward') node.type === 'actionTaker' ? moveActionTakerBackward(node.id) : moveInfoCollectorBackward(node.id);
       else if (action === 'next' && node.type === 'actionTaker') sendActionTakerToDecisionMaker(node.id, 'next');
       else if (action === 'previous' && node.type === 'actionTaker') sendActionTakerToDecisionMaker(node.id, 'previous');
       else if (action === 'delete') requestDeleteComponent(node.id, node.type);
       // 'edit' is dispatched by the node itself (handleOpen), not from here, since it
       // needs the node's own isModalOpen state.
     };
     document.addEventListener('keydown', handleDocumentKeyDown);
     return () => document.removeEventListener('keydown', handleDocumentKeyDown);
   }, [reactFlowInstance, toggleEnabled, moveActionTakerForward, moveActionTakerBackward,
       moveInfoCollectorForward, moveInfoCollectorBackward, sendActionTakerToDecisionMaker, requestDeleteComponent]);
   ```
   Then, at **every** existing `data: { ..., saveNodeData: saveNodeData }` node-creation site (lines 130, 466, 581, 641, 694, 725, 925, 1163), add the five new callbacks so every Action Taker/Info Collector node's `data` carries them, e.g.:
   ```js
   data: { label: nodeCopy[0].id, nodeData: nodeCopy[0], saveNodeData: saveNodeData,
           toggleEnabled, moveForward: moveActionTakerForward, moveBackward: moveActionTakerBackward,
           sendToDecisionMaker: sendActionTakerToDecisionMaker, requestDelete: requestDeleteComponent },
   ```
   (substitute `moveInfoCollectorForward`/`moveInfoCollectorBackward` and omit `sendToDecisionMaker` for Info Collector creation sites, matching each site's existing `type` check).

3. File: `RulesEditor_v2/src/utils/Utils.js`
   Type: Modify
   Intent: Add two small, reusable helpers: `isMacPlatform()` (detects macOS so Step 2/Step 4's listeners use Cmd+Option instead of Ctrl+Alt, per A-6) and `isEditableFocus(activeElement)` (true when focus is in an `input`/`textarea`/`select`/`contenteditable` element or inside the open MUI `Dialog`'s DOM subtree, per REQ-4.10/REQ-6.2/A-3).
   Why: Both the document-level listener (Step 2) and the node-level keyboard handler (Step 4) need the same platform and focus-suspension checks; placing them once in `Utils.js` (already the shared-helpers file, home of `isTrue`, `getUniqueId`) avoids duplicating the check, consistent with "Prefer existing files, functions, and libraries."
   Suggested Change:
   ```js
   // Added near the other small, stateless helpers (isTrue, getUniqueId):

   // macOS uses Cmd+Option for every shortcut in this feature instead of Ctrl+Alt (A-6).
   export function isMacPlatform() {
     return typeof navigator !== 'undefined' && /Mac|iPod|iPhone|iPad/.test(navigator.platform);
   }

   // True when focus is on a text-editable control, or inside an open MUI Dialog, so the
   // per-action shortcuts and the bare Delete/Backspace key are suspended there (REQ-4.10,
   // REQ-6.2, A-3): the key combination is left to edit the text or operate the dialog.
   export function isEditableFocus(activeElement) {
     if (!activeElement) return false;
     const editableTags = ['INPUT', 'TEXTAREA', 'SELECT'];
     if (editableTags.includes(activeElement.tagName) || activeElement.isContentEditable) return true;
     return !!activeElement.closest('.MuiDialog-root, [role="dialog"]');
   }
   ```

4. File: `RulesEditor_v2/src/nodes/ComponentContextMenu.jsx`
   Type: Create
   Intent: The MUI `Menu`/`MenuItem` list for the fixed Action Taker (7-item) and Info Collector (5-item) sets from requirements.md Sections 2 and 3, with icons, visible shortcut text, `disabled`/`disabledItemsFocusable` handling, and `autoFocusItem` driven by how the menu was opened.
   Why: Realizes design.md's Impacted Components "Create: `ComponentContextMenu.jsx`" and Change Plan step 3; satisfies REQ-2.3/REQ-2.4, REQ-3.3/REQ-3.4, and A-9/A-10 using MUI's built-in wrapping arrow-key navigation, Enter-activation, and Escape-close rather than new code.
   Suggested Change:
   ```jsx
   import Menu from '@mui/material/Menu';
   import MenuItem from '@mui/material/MenuItem';
   import ListItemIcon from '@mui/material/ListItemIcon';
   import ListItemText from '@mui/material/ListItemText';
   import Typography from '@mui/material/Typography';
   import ToggleOnIcon from '@mui/icons-material/ToggleOn';
   import ToggleOffIcon from '@mui/icons-material/ToggleOff';
   import ArrowDownwardIcon from '@mui/icons-material/ArrowDownward';
   import ArrowUpwardIcon from '@mui/icons-material/ArrowUpward';
   import ArrowForwardIcon from '@mui/icons-material/ArrowForward';
   import ArrowBackIcon from '@mui/icons-material/ArrowBack';
   import EditIcon from '@mui/icons-material/Edit';
   import DeleteForeverIcon from '@mui/icons-material/DeleteForever';
   import { isMacPlatform } from '../utils/Utils.js';

   // Visible/aria shortcut text per A-6: Ctrl+Alt+<letter> (Windows/Linux), Cmd+Option+<letter> (macOS).
   const shortcutLabel = (letter) => (isMacPlatform() ? `Cmd+Option+${letter}` : `Ctrl+Alt+${letter}`);

   const MenuRow = ({ icon, label, shortcutLetter, disabled, onClick }) => (
     <MenuItem onClick={onClick} disabled={disabled} disabledItemsFocusable aria-keyshortcuts={shortcutLabel(shortcutLetter)}>
       <ListItemIcon>{icon}</ListItemIcon>
       <ListItemText>{label}</ListItemText>
       <Typography variant="body2" color="text.secondary">{shortcutLabel(shortcutLetter)}</Typography>
     </MenuItem>
   );

   // anchorPosition: {x,y} for a right-click open; anchorEl: the node's DOM element for a
   // keyboard-triggered open. openedByKeyboard drives autoFocusItem (A-9).
   const ComponentContextMenu = ({ open, anchorPosition, anchorEl, openedByKeyboard, onClose, componentType, enabled, availability, onToggleEnabled, onMoveForward, onMoveBackward, onSendToNext, onSendToPrevious, onEdit, onDelete }) => {
     const act = (fn) => () => { fn(); onClose(); };
     return (
       <Menu
         open={open}
         onClose={onClose}
         anchorReference={anchorPosition ? 'anchorPosition' : 'anchorEl'}
         anchorPosition={anchorPosition}
         anchorEl={anchorEl}
         autoFocusItem={openedByKeyboard}
       >
         <MenuRow icon={enabled ? <ToggleOffIcon /> : <ToggleOnIcon />} label={enabled ? 'Disable' : 'Enable'} shortcutLetter="T" onClick={act(onToggleEnabled)} />
         <MenuRow icon={<ArrowDownwardIcon />} label="Move Forward" shortcutLetter="J" disabled={!availability.canMoveForward} onClick={act(onMoveForward)} />
         <MenuRow icon={<ArrowUpwardIcon />} label="Move Backward" shortcutLetter="K" disabled={!availability.canMoveBackward} onClick={act(onMoveBackward)} />
         {componentType === 'actionTaker' && (
           <MenuRow icon={<ArrowForwardIcon />} label="Send to Next Decision Maker" shortcutLetter="L" disabled={!availability.canSendToNext} onClick={act(onSendToNext)} />
         )}
         {componentType === 'actionTaker' && (
           <MenuRow icon={<ArrowBackIcon />} label="Send to Previous Decision Maker" shortcutLetter="H" disabled={!availability.canSendToPrevious} onClick={act(onSendToPrevious)} />
         )}
         <MenuRow icon={<EditIcon />} label="Edit" shortcutLetter="O" onClick={act(onEdit)} />
         <MenuRow icon={<DeleteForeverIcon />} label="Delete" shortcutLetter="D" onClick={act(onDelete)} />
       </Menu>
     );
   };
   export default ComponentContextMenu;
   ```

5. File: `RulesEditor_v2/src/nodes/RuleNode.jsx`
   Type: Modify
   Intent: Add `onContextMenu` (right-click entry point, REQ-2.1/REQ-3.1), a `tabIndex` and `onKeyDown` handler matching `DecisionMakerNode.jsx`'s existing pattern (Menu key/Shift+F10 to open the menu with keyboard focus, per-action shortcuts when the node itself has DOM focus, and `Delete`/`Backspace` interception via `e.stopPropagation()` before routing to `requestDelete`), and render `ComponentContextMenu`, branching on `type` for the two Send items.
   Why: Realizes design.md's Impacted Components line for `RuleNode.jsx` and Change Plan steps 3–5; reuses the exact `e.stopPropagation()` technique `DecisionMakerNode.jsx`'s `handleKeyPress` (lines 95–107) already uses so ReactFlow's built-in `deleteKeyCode` never fires for a selected Action Taker/Info Collector.
   Suggested Change:
   ```jsx
   // New imports alongside the existing ones:
   import ComponentContextMenu from './ComponentContextMenu';
   import { getActionTakerMenuAvailability, getInfoCollectorMenuAvailability } from '../utils/ReactFlowUtils.js';
   import { useNodes, useEdges } from 'reactflow'; // useEdges already imported; add useNodes
   import { isEditableFocus } from '../utils/Utils.js';

   // Inside RuleNode, alongside existing state:
   const [menuState, setMenuState] = useState({ open: false, anchorPosition: null, openedByKeyboard: false });
   const nodeRef = useRef();
   const nodes = useNodes(); // edges already available via existing `edges = useEdges()`

   const availability = type === 'actionTaker'
     ? getActionTakerMenuAvailability({ id, parentNode: data.parentNode }, nodes, edges)
     : getInfoCollectorMenuAvailability({ id }, nodes, edges);

   const openMenuAtPoint = (x, y) => setMenuState({ open: true, anchorPosition: { top: y, left: x }, openedByKeyboard: false });
   const openMenuByKeyboard = () => setMenuState({ open: true, anchorPosition: null, openedByKeyboard: true });
   const closeMenu = () => { setMenuState((s) => ({ ...s, open: false })); nodeRef.current?.focus(); }; // REQ-4.6: Escape returns focus to the node

   const handleContextMenu = (e) => {
     e.preventDefault();
     openMenuAtPoint(e.clientX, e.clientY); // REQ-2.1/REQ-3.1; reopens if already open (Section 2/3 Input Handling row a)
   };

   const handleKeyDown = (e) => {
     if (isEditableFocus(document.activeElement)) return; // REQ-4.10/A-3: never when the edit dialog or a text field has focus
     if (e.key === 'ContextMenu' || (e.key === 'F10' && e.shiftKey)) {
       e.preventDefault();
       openMenuByKeyboard(); // REQ-4.2/REQ-4.3
     } else if (e.key === 'Delete' || e.key === 'Backspace') {
       e.stopPropagation(); // matches DecisionMakerNode.jsx's existing technique: block ReactFlow's deleteKeyCode
       data.requestDelete(id, type); // REQ-6.2/REQ-6.3: confirmation first, menu closes if open
       setMenuState((s) => ({ ...s, open: false }));
     }
   };

   // ... in the JSX, add to the existing outer <div>:
   // <div className={...} tabIndex={0} ref={nodeRef} onDoubleClick={handleOpen}
   //      onContextMenu={handleContextMenu} onKeyDown={handleKeyDown} ...>
   //   ...existing content...
   //   <ComponentContextMenu
   //     open={menuState.open} anchorPosition={menuState.anchorPosition} anchorEl={nodeRef.current}
   //     openedByKeyboard={menuState.openedByKeyboard} onClose={closeMenu}
   //     componentType={type} enabled={checkNodeEnableStatus} availability={availability}
   //     onToggleEnabled={() => data.toggleEnabled(id)}
   //     onMoveForward={() => data.moveForward(id)} onMoveBackward={() => data.moveBackward(id)}
   //     onSendToNext={() => data.sendToDecisionMaker(id, 'next')} onSendToPrevious={() => data.sendToDecisionMaker(id, 'previous')}
   //     onEdit={handleOpen} onDelete={() => data.requestDelete(id, type)}
   //   />
   // </div>
   ```

6. File: `RulesEditor_v2/src/DialogBox.jsx`
   Type: Modify
   Intent: Add `autoFocus` to the `Cancel` button (`ConfirmationBox`'s first `Button`, line 25) so it receives default focus (REQ-6.4, REQ-6.10). No new prop is added to the component's public API; the Action-Taker/Info-Collector-specific message text is already supplied by the caller via the existing `message` prop (Step 2's `requestDeleteComponent`), per design.md's note that this is "a caller-side change, not a `DialogBox.jsx` API change."
   Why: Realizes design.md's Impacted Components line for `DialogBox.jsx`. Per design.md's Backwards Compatibility and Risks sections, this also changes default focus for the two existing callers ("new rule", "open rule" confirmations); both are "are you sure you want to discard/replace current changes" prompts where `Cancel` is already the safer default, so no existing behaviour is relied upon.
   Suggested Change:
   ```jsx
   <Button onClick={props.onClose} color="primary" autoFocus>
       Cancel
   </Button>
   ```

7. File: `RulesEditor_v2/src/utils/ReactFlowUtils.test.js`
   Type: Create
   Intent: Unit tests for `getActionTakerMenuAvailability`, `getInfoCollectorMenuAvailability`, `getNextDecisionMaker`/`getPreviousDecisionMaker`, and `swapChainNeighbors`, following the plain-Jest, pure-function-input/output style of the existing `src/utils/Marshaller.test.js` (no React Testing Library needed, since these are non-React helpers).
   Why: Realizes design.md's Validation Strategy "Unit" bullet, and is the regression check for the Risk "Move Forward/Backward and Send-to-Decision-Maker edge rewiring could produce an inconsistent graph."
   Suggested Change:
   ```js
   import { getActionTakerMenuAvailability, getInfoCollectorMenuAvailability, getNextDecisionMaker, getPreviousDecisionMaker, swapChainNeighbors } from './ReactFlowUtils.js';

   // Minimal node/edge fixtures mirroring RuleEditor.js's own shapes (id, type, parentNode).
   const dm = (id) => ({ id, type: 'decisionMaker' });
   const at = (id, parentNode) => ({ id, type: 'actionTaker', parentNode });

   test('Move Forward is unavailable for the last Action Taker in its Decision Maker (A-7)', () => {
     const nodes = [dm('DM1'), at('W', 'DM1'), at('X', 'DM1')];
     const edges = [
       { source: 'DM1', sourceHandle: 'DM_to_AT_DM1', target: 'W' },
       { source: 'W', target: 'X' },
     ];
     expect(getActionTakerMenuAvailability(nodes[2], nodes, edges).canMoveForward).toBe(false);
     expect(getActionTakerMenuAvailability(nodes[1], nodes, edges).canMoveForward).toBe(true);
   });

   test('Send to Next/Previous Decision Maker is unavailable with no such neighbour (A-1)', () => {
     const nodes = [dm('DM1'), at('X', 'DM1')];
     const edges = [{ source: 'DM1', sourceHandle: 'DM_to_AT_DM1', target: 'X' }];
     const availability = getActionTakerMenuAvailability(nodes[1], nodes, edges);
     expect(availability.canSendToNext).toBe(false);
     expect(availability.canSendToPrevious).toBe(false);
   });

   test('swapChainNeighbors reproduces the REQ-2.6 worked example (A,B,C,D,E -> A,B,D,C,E)', () => {
     const nodes = ['A', 'B', 'C', 'D', 'E'].map((id) => at(id, 'DM1'));
     let n = 0;
     const getEdgeId = () => `e${n++}`;
     const edges = [
       { source: 'A', target: 'B' }, { source: 'B', target: 'C' },
       { source: 'C', target: 'D' }, { source: 'D', target: 'E' },
     ];
     const updated = swapChainNeighbors('C', 'D', nodes, edges, getEdgeId);
     const order = ['A', 'B', 'D', 'C', 'E'];
     for (let i = 0; i < order.length - 1; i++) {
       expect(updated.some(e => e.source === order[i] && e.target === order[i + 1])).toBe(true);
     }
   });
   ```

8. File: `webapps/RuleEditor_v2.zip`
   Type: Modify
   Intent: Rebuild the distro artifact from the updated `RulesEditor_v2/src` via `deploy.bat` (`npm run build` then zip), and commit the rebuilt zip in the same change.
   Why: Realizes design.md's Impacted Components "Build artifact" line and Change Plan step 6; per `.github/instructions/rulesengine.instructions.md`, "a change to editor source is not finished until its zip ... is rebuilt and committed. The distro build ships the committed zip, not the source."

9. File: `docs/rulesengine/editors-and-ui.md`
   Type: Modify
   Intent: Add a row to the "Inside the React editor" file table (the existing table listing `src/RuleEditor.js`, `src/utils/ReactFlowUtils.js`, `src/nodes/`, etc.) describing `src/nodes/ComponentContextMenu.jsx`'s responsibility.
   Why: Realizes design.md's Change Plan step 7 and the `rulesengine-react-editor` skill's "After any change" instruction to update this doc when a change affects how the editor works.
   Suggested Change:
   ```markdown
   | `src/nodes/ComponentContextMenu.jsx` | The right-click/keyboard context menu for Action Taker and Info Collector nodes: enable/disable, move, send to another Decision Maker, edit, delete |
   ```

# Inline Comment Requirements
- In `ReactFlowUtils.js`'s new helpers, comment *why* each edge-handle string (`DM_to_AT_`, `DM_to_DM_`) is checked — i.e., that it distinguishes a Decision-Maker-sourced edge from a plain Action-Taker/Info-Collector chain edge — not merely that the check exists; this mirrors the one undocumented convention the rest of the file relies on without explanation.
- In `RuleEditor.js`'s `sendActionTakerToDecisionMaker`, comment why the node is updated in place instead of replaced (the Alternatives Considered rationale: no drag-related transient state to reconcile), so a future reader does not "fix" it to match `onNodeDragStop`'s replica-node approach.
- In `RuleNode.jsx`'s `handleKeyDown`, comment why `e.stopPropagation()` on `Delete`/`Backspace` must happen before anything else in that branch — it is the only thing stopping ReactFlow's built-in `deleteKeyCode` from firing, exactly as `DecisionMakerNode.jsx`'s existing `handleKeyPress` already relies on.
- In `Utils.js`'s `isEditableFocus`, comment that it must cover both a plain form field and the open MUI `Dialog`'s subtree, since the edit dialog's fields are not all native `input`/`textarea` elements.
- Do not add comments that merely restate a line of code (e.g., "// disable the button" on `disabled={!availability.canMoveForward}`); comment only the non-obvious branches above.

# Verification Steps
- Command/Test: `cd RulesEditor_v2 && npm test -- --watchAll=false ReactFlowUtils` (CRA/Jest, non-interactive)
  Expected: the new `ReactFlowUtils.test.js` cases pass, including the Move Forward/Backward boundary cases (A-7), the Send-to-Decision-Maker no-neighbour case (A-1), and the REQ-2.6 worked-example assertion. Verifies requirements REQ-2.6–REQ-2.9, REQ-3.6–REQ-3.7 and design.md's Validation Strategy "Unit" item.
- Command/Test: `cd RulesEditor_v2 && npm test -- --watchAll=false` (full existing Jest suite, e.g. `Marshaller.test.js`, `App.test.js`)
  Expected: all pre-existing tests still pass unchanged — regression check that nothing in `ReactFlowUtils.js`/`Utils.js`'s existing exports was altered.
- Manual Check: `cd RulesEditor_v2 && npm start`, open a rule with at least two Decision Makers each with two Action Takers, and at least two Info Collectors; right-click an Action Taker.
  Expected: the context menu opens at the click point with no item focused until an arrow key is pressed (REQ-2.3, A-9); all seven items appear in the fixed order with their icons and shortcut text (REQ-2.4).
- Manual Check: with exactly one Action Taker selected and canvas focus, press the Menu key (or Shift+F10).
  Expected: the menu opens with "Enable"/"Disable" focused immediately (REQ-2.3/REQ-4.3, A-9); Arrow Down/Up moves focus through all seven items, wrapping, including any greyed-out item (REQ-4.4, A-10); Enter activates only an enabled item; Escape closes the menu and returns focus to the node (REQ-4.5/REQ-4.6).
- Manual Check: select an Action Taker that is last in its Decision Maker, open its menu.
  Expected: "Move Forward" is greyed out but still reachable by Arrow Down/Up (A-7, A-10); its shortcut (Ctrl+Alt+J / Cmd+Option+J) has no effect either, per REQ-4.9's "no effect when ... greyed out."
- Manual Check: perform "Send to Next Decision Maker" on an Action Taker, then reopen the Decision Maker it moved to.
  Expected: it appears after the existing last Action Taker there (REQ-2.8 worked example); its old Decision Maker's remaining Action Takers keep their relative order; use the existing Auto Arrange action and confirm positions update to match the new order.
- Manual Check: with the edit dialog open, or a text field (e.g. a parameter field) focused, press Ctrl+Alt+D / Cmd+Option+D and separately press `Delete`.
  Expected: no effect on the canvas in either case; the key combination edits the focused text field as normal (REQ-4.10, REQ-6.2, A-3).
- Manual Check: with an Info Collector selected and no dialog/field focused, press `Delete`, then separately open its context menu and click "Delete".
  Expected: both open the same `Confirm` dialog with the message "Are you sure you want to delete this Info Collector?", `Cancel` first and focused by default (REQ-6.4, A-12); pressing Escape or clicking `Cancel` leaves the component selected and unchanged (REQ-6.5); choosing `OK` removes it, leaves no component selected afterward (REQ-6.7, A-11), and reconnects its neighbours (REQ-6.6).
- Manual Check: open the equivalent deletion confirmation for a Decision Maker (unchanged by this work).
  Expected: its own `window.confirm`-based flow in `DecisionMakerNode.jsx` is unaffected — confirms this change did not touch Decision Maker deletion, per design.md's Root Cause note and Scope.
- Manual Check: after performing at least one Move Forward, one Send to Next Decision Maker, and one Delete, save the rule (existing Save flow), then reopen it (or inspect via `GetRuleXML`/`exportRule.sh`).
  Expected: the reopened rule's component order and connections match what was left on the canvas, and the GWT editor (`RulesEditor/`) still opens and runs the same rule unchanged — regression check from design.md's Validation Strategy and Risks ("Other Systems: none").
- Manual Check: with "new rule" or "open rule" in progress and unsaved changes present, trigger the existing confirmation dialogs.
  Expected: `Cancel` now receives default focus in both (REQ-6.4/REQ-6.10 side effect noted in design.md's Backwards Compatibility); both confirmations otherwise behave as before.
- Manual Check: with a screen reader (or the browser's accessibility inspector) focused on a menu item.
  Expected: the item's shortcut is announced via `aria-keyshortcuts` alongside its visible label (REQ-4.7).
- Command: `cd RulesEditor_v2 && deploy.bat` (Windows) — rebuilds and copies `webapps/RuleEditor_v2.zip`.
  Expected: build succeeds with no new console errors/warnings from `react-scripts build`; commit the updated `webapps/RuleEditor_v2.zip` alongside the source change, then open `/odsadmin/rule_editor_v2/index.html` and repeat the "open, edit, save, export, import a rule" check from the `rulesengine-react-editor` skill.

# Rollback Plan
- The entire change is isolated to `RulesEditor_v2/src` (`ReactFlowUtils.js`, `RuleEditor.js`, `Utils.js`, the new `ComponentContextMenu.jsx` and its test, `DialogBox.jsx`, `RuleNode.jsx`) and the rebuilt `webapps/RuleEditor_v2.zip`; no database, Java, or `rulesTranslator.jsp` change exists to undo.
- To roll back: revert the commit containing these file changes and the rebuilt `webapps/RuleEditor_v2.zip` (reverting the zip is required — an installation runs the committed zip, not the source). First restore `webapps/RuleEditor_v2.zip` to its pre-change contents if a hotfix is needed faster than reverting source.
- No data migration or re-save of existing rules is needed either direction: a rule saved after using the new menu/shortcuts has the same `Sequence`-derived XML shape a hand-arranged rule would, per design.md's Rollback and Compatibility note.
- If only the deletion-confirmation behavior change proves problematic post-release, the narrowest partial rollback is to remove `'Backspace'`/`'Delete'` handling from `RuleNode.jsx`'s `handleKeyDown` (restoring ReactFlow's immediate `deleteKeyCode` deletion for these two node types) while leaving the context menu and other shortcuts in place; this is also the mitigation design.md's Risks section names if `e.stopPropagation()` does not reliably stop ReactFlow's listener in some browser.

# References
- Design: /work/resources/inputs/artifacts/design.md
- Requirements: /work/resources/inputs/artifacts/requirements.md
- Analysis: none
- Source: /work/resources/inputs/source (reviewed: `RulesEditor_v2/src/RuleEditor.js`, `src/nodes/RuleNode.jsx`, `src/nodes/ActionTakerNode.jsx`, `src/nodes/InfoCollectorNode.jsx`, `src/nodes/DecisionMakerNode.jsx`, `src/DialogBox.jsx`, `src/utils/ReactFlowUtils.js`, `src/utils/Marshaller.js`, `src/utils/Marshaller.test.js`, `src/utils/Utils.js`, `src/Sidebar.js`, `package.json`, `deploy.bat`, `AGENTS.md`, `.github/instructions/rulesengine.instructions.md`, `.github/skills/rulesengine-react-editor/SKILL.md`, `docs/rulesengine/editors-and-ui.md`)
- External: none
- MCP: none used
