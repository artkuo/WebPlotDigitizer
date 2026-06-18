# Repository Dataset Selection

This document describes how the dataset selection behavior works when adding a new dataset in WebPlotDigitizer, and how to change it so that the newly created dataset is automatically selected.

## Existing Behavior

1. **Triggering the Popup**: The "Add Dataset" button in the UI triggers `wpd.dataSeriesManagement.showAddDataset()` in [datasetManagement.js](file:///Users/artkuo/GitProjects/WebPlotDigitizer/javascript/controllers/datasetManagement.js#L49-L59).
2. **Showing the Popup**: `showAddDataset()` determines a default unique name (e.g., "Dataset 0"), sets the value of `#add-single-dataset-name-input`, and opens the `add-dataset-popup` modal.
3. **Handling the Submission**: When the user submits, `addSingleDataset()` is called:
   - It immediately closes the popup with `wpd.popup.close('add-dataset-popup')`.
   - It validates that the dataset name is unique using `datasetWithNameExists()`. If the name already exists, an error message is shown, and the popup is reopened.
   - It creates a new `wpd.Dataset` instance (`ds`), sets its name, and adds it to the plot data, current file, and current page.
   - It calls `wpd.tree.refreshPreservingSelection()`.
   - It dispatches the `"wpd.dataset.add"` event.

### Why Selection is Preserved
Inside [tree.js](file:///Users/artkuo/GitProjects/WebPlotDigitizer/javascript/widgets/tree.js#L550-L558), `refreshPreservingSelection` retrieves the currently selected path using `treeWidget.getSelectedPath()`, rebuilds/renders the tree structure, and then re-selects the previously selected path. Because of this, the new dataset is added to the sidebar tree, but the user's focus/selection remains on whatever dataset was selected *before* they opened the popup.

---

## Proposed Changes

To automatically select the newly added dataset upon creation, the final step in `addSingleDataset()` needs to be modified.

### Files to Modify

#### [MODIFY] [datasetManagement.js](file:///Users/artkuo/GitProjects/WebPlotDigitizer/javascript/controllers/datasetManagement.js)

In `addSingleDataset()`, replace the call to `wpd.tree.refreshPreservingSelection()` with a manual tree refresh followed by selecting the path of the new dataset:

```diff
-        wpd.tree.refreshPreservingSelection();
+        wpd.tree.refresh();
+        wpd.tree.selectPath("/" + wpd.gettext("datasets") + "/" + ds.name);
```

Similarly, in `addMultipleDatasets()`, track the first new dataset that gets created, and select it after refreshing:

```diff
+            let firstNewDataset = null;
             while (i < dsCount) {
                 let dsName = prefix + idx;
                 if (!datasetWithNameExists(dsName)) {
                     let ds = new wpd.Dataset();
                     ds.name = dsName;
+                    if (firstNewDataset == null) {
+                        firstNewDataset = ds;
+                    }
...
-            wpd.tree.refreshPreservingSelection();
+            wpd.tree.refresh();
+            if (firstNewDataset != null) {
+                wpd.tree.selectPath("/" + wpd.gettext("datasets") + "/" + firstNewDataset.name);
+            }
```

This ensures that:
1. The tree is updated to include the new dataset(s).
2. `selectPath` is invoked with the tree path of the new dataset (or the first of the multiple added datasets), changing the active dataset selection and updating the corresponding UI elements (such as the point-acquisition sidebars and graphics tools).

