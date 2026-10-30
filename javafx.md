# JavaFX guidelines

Read before building any JavaFX UI. The general Java rules are in `java.md`.

- USE THIS `../../luvml/luvjfx/README.md` - this is nice terse code looks like html luvml but human in the loop is lacking; for one off projects this is ok.
- Other option is to make a pure fxml based components as individual maven projects and then add them back in scenebuilder and also re-use/wire them together again as fxml maven projectl advantage is you can see it in scenebuilder and graphically make edits graphically; this is a good AI and human in loop; for projects which require detailed designed components, this is better.

- JavaFX UI :

	- Prefer JavaFx for UI, defer Swing, and strongly defer AWT, unless until absolutely necessary to compensate missing features in JavaFx or it is much much more simpler.

	- There is a javafx bug, you cannot load a shaded jar from the Application main, you need to create one more main (call it Main). This Main will call the Application main method, only then reflection etc. for javafx work properly, especially in a shaded jar, otherwise it will complain javafx modules not loaded properly etc. etc.

	- For JavaFx freely use all modern libraries like controlfx and any library out there which is useful for the purpose and freely available.

	- Be mindful of the javafx version, in java is 25, javafx can be 22, 23 ... but keep it 25 for consistency. (we will upgrade once the next LTS is released)

	- You should take some effort and draw an icon for the project, either it can be done as a raster/vector through java code, or a vector through xml (svg), (but don't do base64 image it consumes too much space and token) or anything else is also fine, but let not all projects be boring same tea cup and not distinguishable. This icon should be used by the window of the main app, and systemtray (if being used). You might find the need of batik to convert from svg to png or java graphics image object, in that case just draw in a java graphics / canvas directly! no need of svg also. But avoid png please.
		- **A drawn icon must reach the Stage through JavaFX's image loader, NOT as a `Canvas.snapshot(...)` `WritableImage`.** On Windows, JavaFX silently ignores a `WritableImage` in `stage.getIcons()`: the title bar and taskbar show the generic Windows window glyph, with no exception, no warning, and `getIcons()` still reporting valid images full of the right pixels. Draw on the Canvas as above, then pass the pixels through PNG in memory and load that: `ImageIO.write(bufferedImage, "png", bytes)` → `new Image(new ByteArrayInputStream(bytes.toByteArray()))`. "Avoid png" above means do not SHIP png files; a PNG made in memory from the drawing is the required route, not a violation. The system tray is not affected, since it takes an AWT image. Worked implementation: `../../cfucli/cfucli/app/src/main/java/cfucli/app/Icons.java` (`windowIcons` → `loaded`). Found 2026-09-26, and it had been broken since that project's first icon - every earlier check passed.
		- **Verify the icon in a REAL title bar, zoomed in - not by rendering the images to a file.** Rendering every size to a PNG sheet and looking at it proved the drawing was perfect, and said nothing about the defect, because the loss happens between `getIcons()` and the window. At normal screenshot size a 16px default glyph and a 16px logo are both "a small icon"; only an enlarged crop of the title bar showed which one it was. When a path fails silently, put a known-good control beside it: a window with a file-loaded icon next to one with the drawn icon (`IconWindowProbe` in the same project) settled in one screenshot what reading the code and the logs could not.

	- if you ever implement or use clipboard or systemtray some feature which needs java swing or awt, even in those cases prefer to find a javafx solution, if it is not there, no problem AI can use swing/awt in such rare cases, but prefer javafx. Consuider using - https://github.com/dustinkredmond/FXTrayIcon

	- Sometimes we are selecting a feature, that the java app should minimize (to system tray) on closing the application window. Such apps are actually intended to run a as a single instane. For such app, if another instance is run it should immediately call the already running instance to popup itself from the system tray ... such a mechanism may be desired, a standard library for this can be made and then consequtively reused.

- **Accessibility — keyboard-first navigation**: some of us have a hand/nerve condition that makes combination keys and mouse use painful, single keys are much easier. So javafx UIs should support keyboard-first navigation by default: Tab to move focus between controls, single-key shortcuts to trigger actions (e.g. Y/N, or jump straight to a button) and Esc to cancel/close, plus a status-bar line listing the shortcuts available — like old MS-DOS/Windows-95 era TUI conventions, or how IDEs still do it today. Nice-to-have, not compulsory, but lean into it by default.
	- **Before binding a key:** "single-key shortcuts" above is true ONLY where nothing is competing for that key. A window with a search/filter/name box owes every letter and digit to typing, so its actions get F-keys; a pure menu with no text input may use letters. Watch for the JavaFX mnemonic trap (`new CheckBox("_Auxiliary")` stops requiring Alt after the first Alt press in the Scene, then eats the bare A), which is not guessable and has already shipped once.
	- **A window that GRANTS something (approve, extend, allow) must never be answered by a stray keystroke.** It pops up on top and takes the focus while the user is typing elsewhere: a word with an "a" in it once approved a gate request. Ignore every key for the first ~2 seconds (buttons disabled, a visible countdown), then approve on an F-key, never a letter; Esc denies. Worked in leasegate-ui (`Arming`, `ApprovalWindow`), 2026-10-20.

- Debug JavaFX Scene
	- Consider adding ScenicView, but adding `ScenicView.show(scene);` BEFORE showing the stage adding ( you will ned to `import org.scenicview.ScenicView;` and include dependency `org.scenicview:scenicview:1.0.0` in pom ) keep all this commented out in general, the user can enable need on basis both in pom and ApplicationMain

- For a resident JavaFX app (tray, daemon), also read `resident-apps-memory.md`: never poll with an endless `Timeline`.

- Useful JavaFX libraries and apps
	- Scenic View — inspect the state of a running JavaFX app: https://github.com/JonathanGiles/scenic-view
	- MDFX — markdown rendering: https://github.com/JPro-one/JPro-Platform/tree/main/jpro-mdfx
	- FXRibbon: https://github.com/dukke/FXRibbon
	- StagePro — menu in the title bar: https://github.com/techsenger/stagepro
	- tiwulfx-dock — docking panes: https://github.com/panemu/tiwulfx-dock
	- FXTrayIcon — system tray: https://github.com/dustinkredmond/FXTrayIcon
	- Others worth knowing: ControlsFX, FXSampler, GemsFX, CSSFX, RichTextFX, TerminalFX, DataFX. JavaFX spreadsheet (scalable-components.com) is paid.
