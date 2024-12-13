============
Directory Structure:
============
├── .gradle
├── .idea
├── .kotlin
├── build
├── gradle
├── src
│   ├── main
│   │   ├── kotlin
│   │   │   └── Main.kt
│   └── test
│       ├── kotlin
├── build.gradle.kts
├── gradle.properties
├── oneFile.md
└── settings.gradle.kts

============
oneFile.md : /Users/chetan.gupta/Desktop/ch8n/rough/1fileKt/oneFile.md
============
============
Directory Structure:
============
├── .gradle
├── .idea
├── .kotlin
├── build
├── gradle
├── src
│   ├── main
│   │   ├── kotlin
│   │   │   └── Main.kt
│   └── test
│       ├── kotlin
├── build.gradle.kts
├── gradle.properties
├── oneFile.md
└── settings.gradle.kts

============
build.gradle.kts : /Users/chetan.gupta/Desktop/ch8n/rough/1fileKt/build.gradle.kts
============
plugins {
    kotlin("jvm") version "2.0.10"
}

group = "dev.ch8n"
version = "1.0-SNAPSHOT"

repositories {
    mavenCentral()
}

dependencies {
    testImplementation(kotlin("test"))
}

tasks.test {
    useJUnitPlatform()
}

============
settings.gradle.kts : /Users/chetan.gupta/Desktop/ch8n/rough/1fileKt/settings.gradle.kts
============
rootProject.name = "1fileKt"



============
gradle.properties : /Users/chetan.gupta/Desktop/ch8n/rough/1fileKt/gradle.properties
============
kotlin.code.style=official


============
Main.kt : /Users/chetan.gupta/Desktop/ch8n/rough/1fileKt/src/main/kotlin/Main.kt
============
import java.io.File
import java.nio.file.FileSystems
import java.nio.file.PathMatcher


class OneFile() {
    fun execute(
        rootDirectory: File,
        ignorePatterns: List<String>,
        outfileName: String = "oneFile.md"
    ): File {
        if (!rootDirectory.isDirectory) throw IllegalArgumentException("The provided path is not a directory.")

        val excludePatterns = ignorePatterns.filter { !it.startsWith("!") }
            .map { adjustPattern(it) }
        val includePatterns = ignorePatterns.filter { it.startsWith("!") }
            .map { adjustPattern(it.removePrefix("!")) }

        val excludeMatchers = excludePatterns.map { pattern ->
            FileSystems.getDefault().getPathMatcher("glob:$pattern")
        }

        val includeMatchers = includePatterns.map { pattern ->
            FileSystems.getDefault().getPathMatcher("glob:$pattern")
        }

        var fileCount = 0
        var directoryCount = 0
        var totalLines = 0
        var totalWords = 0

        val outputFile = File(rootDirectory, outfileName)
        outputFile.bufferedWriter().use { writer ->
            // Write directory structure
            writer.write("============\n")
            writer.write("Directory Structure:\n")
            writer.write("============\n")
            writer.write(
                getDirectoryStructure(
                    rootDirectory.toPath(),
                    rootDirectory.toPath(),
                    excludeMatchers,
                    includeMatchers
                )
            )
            writer.write("\n")

            // Write file contents
            rootDirectory.walkTopDown()
                .filter {
                    it.isFile &&
                            it.absolutePath != outputFile.absolutePath &&
                            !isExcluded(it.toPath(), rootDirectory.toPath(), excludeMatchers, includeMatchers)
                }
                .forEach { file ->
                    writer.write("============\n")
                    writer.write("${file.name} : ${file.absolutePath}\n")
                    writer.write("============\n")
                    val content = file.readText()
                    writer.write(content)
                    writer.write("\n\n")

                    fileCount++
                    totalLines += content.lineSequence().count()
                    totalWords += content.split(Regex("\\s+")).count()
                }

            // Count directories excluding the root
            directoryCount = rootDirectory.walkTopDown()
                .filter {
                    it.isDirectory &&
                            it != rootDirectory &&
                            !isExcluded(it.toPath(), rootDirectory.toPath(), excludeMatchers, includeMatchers)
                }
                .count()

            // Write summary
            writer.write("========\n")
            writer.write("Summary\n")
            writer.write("========\n")
            writer.write("Repository: ${rootDirectory.name}\n")
            writer.write("Files analyzed: $fileCount\n")
            writer.write("Directories analyzed: $directoryCount\n")
            writer.write("Total lines of content: $totalLines\n")
            writer.write("Total words: $totalWords\n")
            if (ignorePatterns.isNotEmpty()) {
                writer.write("Excluded/Inclusion patterns:\n")
                ignorePatterns.forEach { pattern ->
                    writer.write("- $pattern\n")
                }
            }
        }

        return outputFile
    }

    /**
     * Adjusts the pattern to ensure directories are fully excluded by appending '**' if needed.
     */
    fun adjustPattern(pattern: String): String {
        return if (pattern.endsWith("/")) {
            "${pattern}**"
        } else {
            pattern
        }
    }

    /**
     * Generates a directory structure string similar to the `tree` command,
     * respecting the exclusion and inclusion patterns.
     */
    fun getDirectoryStructure(
        currentPath: java.nio.file.Path,
        rootPath: java.nio.file.Path,
        excludeMatchers: List<PathMatcher>,
        includeMatchers: List<PathMatcher>,
        prefix: String = ""
    ): String {
        val builder = StringBuilder()
        val files = currentPath.toFile().listFiles()?.sortedWith(compareBy({ !it.isDirectory }, { it.name }))
            ?: return builder.toString()

        files.forEachIndexed { index, file ->
            val isLast = index == files.lastIndex
            val branch = if (isLast) "└── " else "├── "
            val newPrefix = prefix + branch
            val childPrefix = prefix + if (isLast) "    " else "│   "

            // Get the relative path from the root directory
            val relativePath = rootPath.relativize(file.toPath())

            if (isExcluded(file.toPath(), rootPath, excludeMatchers, includeMatchers)) {
                // Skip excluded files/directories
                return@forEachIndexed
            }

            builder.append("$newPrefix${file.name}\n")
            if (file.isDirectory) {
                builder.append(
                    getDirectoryStructure(
                        file.toPath(),
                        rootPath,
                        excludeMatchers,
                        includeMatchers,
                        childPrefix
                    )
                )
            }
        }
        return builder.toString()
    }

    /**
     * Determines whether a given path should be excluded based on the excludeMatchers and includeMatchers.
     * Inclusion patterns (!patterns) override exclusion patterns.
     */
    fun isExcluded(
        path: java.nio.file.Path,
        rootPath: java.nio.file.Path,
        excludeMatchers: List<PathMatcher>,
        includeMatchers: List<PathMatcher>
    ): Boolean {
        val relativePath = rootPath.relativize(path).toString().replace(File.separatorChar, '/')

        // Check inclusion patterns first
        if (includeMatchers.any { matcher -> matcher.matches(FileSystems.getDefault().getPath(relativePath)) }) {
            return false
        }

        // Then check exclusion patterns
        return excludeMatchers.any { matcher -> matcher.matches(FileSystems.getDefault().getPath(relativePath)) }
    }
}


fun main() {
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
}



========
Summary
========
Repository: 1fileKt
Files analyzed: 4
Directories analyzed: 10
Total lines of content: 229
Total words: 607
Excluded/Inclusion patterns:
- merged_content.md
- **/build/
- build/
- gradlew
- gradlew.bat
- .gradle/
- gradle/
- **/resources
- .idea/
- .kotlin/
- .gitignore
- .git/
- .git
- *.iws
- *.iml
- *.ipr
- out/
- **/src/main/**/out/
- **/src/test/**/out/
- *.classpath
- *.factorypath
- .apt_generated
- .project
- .settings
- .springBeans
- .sts4-cache
- bin/
- **/src/main/**/bin/
- **/src/test/**/bin/
- nbproject/private/
- nbbuild/
- dist/
- nbdist/
- .nb-gradle/
- .vscode/
- .DS_Store


============
build.gradle.kts : /Users/chetan.gupta/Desktop/ch8n/rough/1fileKt/build.gradle.kts
============
plugins {
    kotlin("jvm") version "2.0.10"
}

group = "dev.ch8n"
version = "1.0-SNAPSHOT"

repositories {
    mavenCentral()
}

dependencies {
    testImplementation(kotlin("test"))
}

tasks.test {
    useJUnitPlatform()
}

============
settings.gradle.kts : /Users/chetan.gupta/Desktop/ch8n/rough/1fileKt/settings.gradle.kts
============
rootProject.name = "1fileKt"



============
gradle.properties : /Users/chetan.gupta/Desktop/ch8n/rough/1fileKt/gradle.properties
============
kotlin.code.style=official


============
Main.kt : /Users/chetan.gupta/Desktop/ch8n/rough/1fileKt/src/main/kotlin/Main.kt
============
import java.io.File
import java.nio.file.FileSystems
import java.nio.file.PathMatcher


class OneFile() {

    fun execute(
        rootDirectory: File,
        ignorePatterns: List<String>,
        outfileName: String = "${rootDirectory.name}.md"
    ): File {
        if (!rootDirectory.isDirectory) throw IllegalArgumentException("The provided path is not a directory.")

        val excludePatterns = ignorePatterns.filter { !it.startsWith("!") }
            .map { adjustPattern(it) }
        val includePatterns = ignorePatterns.filter { it.startsWith("!") }
            .map { adjustPattern(it.removePrefix("!")) }

        val excludeMatchers = excludePatterns.map { pattern ->
            FileSystems.getDefault().getPathMatcher("glob:$pattern")
        }

        val includeMatchers = includePatterns.map { pattern ->
            FileSystems.getDefault().getPathMatcher("glob:$pattern")
        }

        var fileCount = 0
        var directoryCount = 0
        var totalLines = 0
        var totalWords = 0

        val outputFile = File("src/main/resources", outfileName)
        outputFile.bufferedWriter().use { writer ->
            // Write directory structure
            writer.write("============\n")
            writer.write("Directory Structure:\n")
            writer.write("============\n")
            writer.write(
                getDirectoryStructure(
                    rootDirectory.toPath(),
                    rootDirectory.toPath(),
                    excludeMatchers,
                    includeMatchers
                )
            )
            writer.write("\n")

            // Write file contents
            rootDirectory.walkTopDown()
                .filter {
                    it.isFile &&
                            it.absolutePath != outputFile.absolutePath &&
                            !isExcluded(it.toPath(), rootDirectory.toPath(), excludeMatchers, includeMatchers)
                }
                .forEach { file ->
                    writer.write("============\n")
                    writer.write("${file.name} : ${file.absolutePath}\n")
                    writer.write("============\n")
                    val content = file.readText()
                    writer.write(content)
                    writer.write("\n\n")

                    fileCount++
                    totalLines += content.lineSequence().count()
                    totalWords += content.split(Regex("\\s+")).count()
                }

            // Count directories excluding the root
            directoryCount = rootDirectory.walkTopDown()
                .filter {
                    it.isDirectory &&
                            it != rootDirectory &&
                            !isExcluded(it.toPath(), rootDirectory.toPath(), excludeMatchers, includeMatchers)
                }
                .count()

            // Write summary
            writer.write("========\n")
            writer.write("Summary\n")
            writer.write("========\n")
            writer.write("Repository: ${rootDirectory.name}\n")
            writer.write("Files analyzed: $fileCount\n")
            writer.write("Directories analyzed: $directoryCount\n")
            writer.write("Total lines of content: $totalLines\n")
            writer.write("Total words: $totalWords\n")
            if (ignorePatterns.isNotEmpty()) {
                writer.write("Excluded/Inclusion patterns:\n")
                ignorePatterns.forEach { pattern ->
                    writer.write("- $pattern\n")
                }
            }
        }

        return outputFile
    }

    /**
     * Adjusts the pattern to ensure directories are fully excluded by appending '**' if needed.
     */
    fun adjustPattern(pattern: String): String {
        return if (pattern.endsWith("/")) {
            "${pattern}**"
        } else {
            pattern
        }
    }

    /**
     * Generates a directory structure string similar to the `tree` command,
     * respecting the exclusion and inclusion patterns.
     */
    fun getDirectoryStructure(
        currentPath: java.nio.file.Path,
        rootPath: java.nio.file.Path,
        excludeMatchers: List<PathMatcher>,
        includeMatchers: List<PathMatcher>,
        prefix: String = ""
    ): String {
        val builder = StringBuilder()
        val files = currentPath.toFile().listFiles()?.sortedWith(compareBy({ !it.isDirectory }, { it.name }))
            ?: return builder.toString()

        files.forEachIndexed { index, file ->
            val isLast = index == files.lastIndex
            val branch = if (isLast) "└── " else "├── "
            val newPrefix = prefix + branch
            val childPrefix = prefix + if (isLast) "    " else "│   "

            // Get the relative path from the root directory
            val relativePath = rootPath.relativize(file.toPath())

            if (isExcluded(file.toPath(), rootPath, excludeMatchers, includeMatchers)) {
                // Skip excluded files/directories
                return@forEachIndexed
            }

            builder.append("$newPrefix${file.name}\n")
            if (file.isDirectory) {
                builder.append(
                    getDirectoryStructure(
                        file.toPath(),
                        rootPath,
                        excludeMatchers,
                        includeMatchers,
                        childPrefix
                    )
                )
            }
        }
        return builder.toString()
    }

    /**
     * Determines whether a given path should be excluded based on the excludeMatchers and includeMatchers.
     * Inclusion patterns (!patterns) override exclusion patterns.
     */
    fun isExcluded(
        path: java.nio.file.Path,
        rootPath: java.nio.file.Path,
        excludeMatchers: List<PathMatcher>,
        includeMatchers: List<PathMatcher>
    ): Boolean {
        val relativePath = rootPath.relativize(path).toString().replace(File.separatorChar, '/')

        // Check inclusion patterns first
        if (includeMatchers.any { matcher -> matcher.matches(FileSystems.getDefault().getPath(relativePath)) }) {
            return false
        }

        // Then check exclusion patterns
        return excludeMatchers.any { matcher -> matcher.matches(FileSystems.getDefault().getPath(relativePath)) }
    }
}


fun main() {
    val rootDirectoryPath = "/Users/chetan.gupta/Desktop/ch8n/rough/1fileKt"
    val rootDirectory = File(rootDirectoryPath)
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
}



========
Summary
========
Repository: 1fileKt
Files analyzed: 5
Directories analyzed: 10
Total lines of content: 540
Total words: 1367
Excluded/Inclusion patterns:
- merged_content.md
- **/build/
- build/
- gradlew
- gradlew.bat
- .gradle/
- gradle/
- **/resources
- .idea/
- .kotlin/
- .gitignore
- .git/
- .git
- *.iws
- *.iml
- *.ipr
- out/
- **/src/main/**/out/
- **/src/test/**/out/
- *.classpath
- *.factorypath
- .apt_generated
- .project
- .settings
- .springBeans
- .sts4-cache
- bin/
- **/src/main/**/bin/
- **/src/test/**/bin/
- nbproject/private/
- nbbuild/
- dist/
- nbdist/
- .nb-gradle/
- .vscode/
- .DS_Store
