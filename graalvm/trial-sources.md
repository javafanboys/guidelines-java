# Trial programs: complete sources, to reproduce a build from the notes

Built and run 2026-10-30 (see `measurements.md` for the results). Use them as templates: copy a folder, change the package and names. Build each with the per-process script from `install-and-toolchain.md`: `build-native.cmd <folder>` (add `-Dnative.optimization=b` for the quick mode), then run `target/<name>.exe`.

## A. JavaFX window through StaticFX, FXML view, annotation recipes

Layout:

```
hellofx/pom.xml
hellofx/src/main/java/trial/hellofx/{Launcher,HelloFX,GreetingView,GreetingController}.java
hellofx/src/main/resources/trial/hellofx/{greeting.fxml,greeting.css}
```

`pom.xml`:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <groupId>trial</groupId><artifactId>hellofx</artifactId><version>1.0</version><packaging>jar</packaging>
  <properties>
    <maven.compiler.release>25</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <main.class>trial.hellofx.Launcher</main.class>
    <javafx.version>27</javafx.version>
    <jfx.static.version>27-2</jfx.static.version>
    <jfx.static.feature.version>1.1.1</jfx.static.feature.version>
    <reachability.version>1.0.0-RC5</reachability.version>
    <native.plugin.version>1.1.18</native.plugin.version>
    <native.optimization>2</native.optimization>
  </properties>
  <dependencies>
    <dependency><groupId>us.hebi.graalvm</groupId><artifactId>reachability-annotations</artifactId><version>${reachability.version}</version><scope>provided</scope></dependency>
    <dependency><groupId>org.openjfx</groupId><artifactId>javafx-controls</artifactId><version>${javafx.version}</version></dependency>
    <dependency><groupId>org.openjfx</groupId><artifactId>javafx-fxml</artifactId><version>${javafx.version}</version></dependency>
    <dependency><groupId>us.hebi.graalvm</groupId><artifactId>jfx-static-feature</artifactId><version>${jfx.static.feature.version}</version></dependency>
    <dependency><groupId>us.hebi.graalvm</groupId><artifactId>jfx-static-libs</artifactId><version>${jfx.static.version}</version><scope>runtime</scope></dependency>
  </dependencies>
  <build><plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId><artifactId>maven-compiler-plugin</artifactId><version>3.13.0</version>
      <configuration>
        <annotationProcessorPaths><path><groupId>us.hebi.graalvm</groupId><artifactId>reachability-processor</artifactId><version>${reachability.version}</version></path></annotationProcessorPaths>
        <compilerArgs>
          <arg>-Aproject=${project.groupId}/${project.artifactId}</arg>
          <arg>-Areachability.outputFormat=1.0.0</arg>
        </compilerArgs>
      </configuration>
    </plugin>
  </plugins></build>
  <profiles><profile>
    <id>native</id>
    <build><plugins><plugin>
      <groupId>org.graalvm.buildtools</groupId><artifactId>native-maven-plugin</artifactId><version>${native.plugin.version}</version>
      <executions><execution><id>build-native</id><phase>package</phase><goals><goal>compile-no-fork</goal></goals></execution></executions>
      <configuration>
        <imageName>hellofx</imageName>
        <mainClass>${main.class}</mainClass>
        <metadataRepository><enabled>false</enabled></metadataRepository>
        <buildArgs>
          <buildArg>-O${native.optimization}</buildArg>
          <buildArg>--exact-reachability-metadata</buildArg>
        </buildArgs>
      </configuration>
    </plugin></plugins></build>
  </profile></profiles>
</project>
```

`Launcher.java`:

```java
package trial.hellofx;

import javafx.application.Application;

public class Launcher {
    public static void main(String[] args) {
        Application.launch(HelloFX.class, args);
    }
}
```

`HelloFX.java` (the `--exit` mode is the self-test: it proves the FXML controller and its reflective `#clicked` handler work in the image):

```java
package trial.hellofx;

import javafx.application.Application;
import javafx.application.Platform;
import javafx.scene.Scene;
import javafx.scene.control.Button;
import javafx.scene.control.Label;
import javafx.stage.Stage;

public class HelloFX extends Application {

    @Override
    public void start(Stage stage) throws Exception {
        stage.setTitle("HelloFX trial");
        stage.setScene(new Scene(GreetingView.load(), 420, 220));
        if (getParameters().getRaw().contains("--exit")) {
            stage.setOnShown(e -> Platform.runLater(() -> selfTestAndExit(stage)));
        }
        stage.show();
    }

    void selfTestAndExit(Stage stage) {
        var root = stage.getScene().getRoot();
        ((Button) root.lookup(".button")).fire();
        var text = ((Label) root.lookup(".label")).getText();
        if (!text.equals("Clicked 1 times")) {
            System.err.println("self-test failed: " + text);
            System.exit(3);
        }
        Platform.exit();
    }
}
```

`GreetingView.java`:

```java
package trial.hellofx;

import java.io.IOException;
import javafx.fxml.FXMLLoader;
import javafx.scene.Parent;
import us.hebi.graalvm.reachability.annotations.ReachableFxView;

@ReachableFxView
public class GreetingView {
    public static Parent load() throws IOException {
        return FXMLLoader.load(GreetingView.class.getResource("greeting.fxml"));
    }
}
```

`GreetingController.java`:

```java
package trial.hellofx;

import javafx.fxml.FXML;
import javafx.scene.control.Label;

public class GreetingController {
    @FXML Label label;
    int clicks;

    @FXML void clicked() {
        label.setText("Clicked " + (++clicks) + " times");
    }
}
```

`greeting.fxml` (EXPLICIT imports; wildcard imports crash in exact mode, see `traps.md`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?import javafx.scene.control.Button?>
<?import javafx.scene.control.Label?>
<?import javafx.scene.layout.VBox?>

<VBox xmlns:fx="http://javafx.com/fxml" fx:controller="trial.hellofx.GreetingController"
      alignment="CENTER" spacing="16" stylesheets="@greeting.css">
    <Label fx:id="label" text="Hello from FXML"/>
    <Button text="Click me" onAction="#clicked"/>
</VBox>
```

`greeting.css`: `.label { -fx-font-size: 18px; }`

Expected: the build prints `Finished generating 'hellofx'` and the 1.0.0-format metadata for `GreetingController`, the node types, and `greeting.fxml`, `greeting.css`, `greeting.bss`, `greeting*.properties` under `target/classes/META-INF/native-image/reachability-generated/trial/hellofx/`; `cmd /c "hellofx.exe --exit > run.log 2>&1"` exits 0 (exit 3 means the handler did not run; an exception in `run.log` means a registration is missing). Run without `--exit` to see the window.

## B. Class, resource and bundle looked up by computed name

Same pom as A without the JavaFX and StaticFX dependencies, `main.class` `trial.reflect.Main`, `imageName` `reflect`, `native.optimization` `b`. Files:

```
reflect/src/main/java/trial/reflect/{Main,Plugin}.java
reflect/src/main/java/trial/reflect/plugins/{Hello,Bye}.java
reflect/src/main/resources/templates/hello.txt                 (text: template hello)
reflect/src/main/resources/trial/reflect/messages.properties   (text: hello=bundle hello)
```

`Plugin` is `public interface Plugin { String greet(); }`; `Hello` and `Bye` are public classes implementing it, returning `"hello"` and `"bye"`. `Main.java` (first build WITHOUT the annotation lines to see the three `Missing...RegistrationError`s, then add them):

```java
package trial.reflect;

import java.util.ResourceBundle;
import trial.reflect.plugins.*;
import us.hebi.graalvm.reachability.annotations.*;
import static us.hebi.graalvm.reachability.annotations.MemberAccess.ALL_DECLARED_CONSTRUCTORS;

@Reachable(
    classes = {Hello.class, Bye.class},
    memberAccess = ALL_DECLARED_CONSTRUCTORS,
    resources = "/templates/*.txt",
    bundles = "messages")
public class Main {

    public static void main(String[] args) throws Exception {
        var mode = args[0];
        var name = args[1];
        switch (mode) {
            case "class" -> {
                var plugin = (Plugin) Class.forName("trial.reflect.plugins." + name).getDeclaredConstructor().newInstance();
                System.out.println("class ok: " + plugin.greet());
            }
            case "resource" -> {
                try (var in = Main.class.getResourceAsStream("/templates/" + name + ".txt")) {
                    System.out.println("resource " + (in == null ? "MISSING" : "ok: " + new String(in.readAllBytes()).trim()));
                }
            }
            case "bundle" -> System.out.println("bundle ok: " + ResourceBundle.getBundle("trial.reflect." + name).getString("hello"));
            default -> System.out.println("usage: class|resource|bundle <name>");
        }
    }
}
```

Run `reflect.exe class Hello`, `class Bye`, `resource hello`, `bundle messages`: each prints `... ok: ...` with the annotation, and throws a `Missing...RegistrationError` naming the entry without it.

## C. FFM downcalls (a CLI that deletes a Windows reserved-name file)

Any program that does `Linker.nativeLinker().downcallHandle(...)` for `jint DeleteFileW(void*)` and `jint GetLastError()` (kernel32, looked up with `SymbolLookup.libraryLookup("kernel32", Arena.global())`). Profile as in `install-and-toolchain.md` plus `--enable-native-access=ALL-UNNAMED`. Without metadata the first delete throws `MissingForeignRegistrationError`; with this file it works:

`src/main/resources/META-INF/native-image/<groupId>/<artifactId>/reachability-metadata.json`

```json
{
  "foreign": {
    "downcalls": [
      { "returnType": "jint", "parameterTypes": ["void*"] },
      { "returnType": "jint", "parameterTypes": [] }
    ]
  }
}
```

To create the test input on Windows from .NET/PowerShell: `[System.IO.File]::WriteAllText("\\?\<folder>\nul","x")`.
