# 1FileKt
Feed your entire codebase to GPTs/LLM. 1 file for your entire codebase ingestion.

# How to use

```kotlin
val directoryPath = "/Users/chetan.gupta/Desktop/ch8n/rough/1fileKt"
    val rootDirectory = File(directoryPath)
    val ignorePatterns = mutableListOf<String>(
        "merged_content.md",
        "**/build/", // Exclude all build directories
        "build/",     // Exclude build directories at root
        "gradlew", "gradlew.bat",
        ".gradle/",   // Exclude .gradle directory and its contents
        "gradle/",    // Exclude gradle directory and its contents
        "**/resources",
        ".idea/",     // Exclude .idea directory and its contents
        ".kotlin/",   // Exclude .kotlin directory and its contents
        ".gitignore", ".git/", ".git", // Exclude git related files/directories
        "*.iws", "*.iml", "*.ipr",
        "out/",             // Exclude out directories and contents
        "**/src/main/**/out/", "**/src/test/**/out/",
        "*.classpath", "*.factorypath",
        ".apt_generated", ".project", ".settings", ".springBeans", ".sts4-cache",
        "bin/",             // Exclude bin directories and contents
        "**/src/main/**/bin/", "**/src/test/**/bin/",
        "nbproject/private/", "nbbuild/", "dist/", "nbdist/", ".nb-gradle/",
        ".vscode/",
        ".DS_Store"
    )
    val oneFile = OneFile()
    val outputFile = oneFile.execute(rootDirectory, ignorePatterns)
    print(outputFile.readText())
```
