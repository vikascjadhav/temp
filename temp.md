{
    // ==========================================
    // Java Language Server
    // ==========================================

    // Full Java language-server functionality.
    "java.server.launchMode": "Standard",

    // Automatically import Java projects when opening the workspace.
    "java.project.importOnFirstTimeStartup": "automatic",

    // Automatically update Java classpath/configuration
    // when pom.xml changes.
    "java.configuration.updateBuildConfiguration": "automatic",

    // ==========================================
    // Java Build / Compilation
    // ==========================================

    // Automatically compile Java sources.
    "java.autobuild.enabled": true,

    // Allow Maven projects to be imported.
    "java.import.maven.enabled": true,

    // Download source JARs for dependency navigation.
    "java.maven.downloadSources": true,

    // ==========================================
    // Generated Sources
    // ==========================================

    // IMPORTANT:
    // Do NOT exclude target/generated-sources from
    // Java Language Server processing.
    //
    // Your OpenAPI/generated Java sources live under:
    //
    // target/generated-sources/...
    //
    // Maven should expose these as source roots.

    // ==========================================
    // Java Navigation
    // ==========================================

    "editor.gotoLocation.multipleDefinitions": "peek",
    "editor.gotoLocation.multipleImplementations": "peek",
    "editor.gotoLocation.multipleReferences": "peek",

    // Show implementation/reference information
    // where supported by the Java language server.
    "java.implementationsCodeLens.enabled": true,
    "java.referencesCodeLens.enabled": true,

    // ==========================================
    // Java Editor
    // ==========================================

    "editor.semanticHighlighting.enabled": true,

    "java.completion.enabled": true,

    // Organize imports automatically on save.
    "java.saveActions.organizeImports": true,

    // ==========================================
    // Search
    // ==========================================

    // Keep generated/build output out of normal
    // text searches.
    //
    // IMPORTANT:
    // This does NOT tell the Java Language Server
    // to ignore generated Java classes.
    "search.exclude": {
        "**/target": true,
        "**/node_modules": true,
        "**/.git": true
    },

    // ==========================================
    // Explorer
    // ==========================================

    // Hide target from the VS Code Explorer if desired.
    // This is purely visual; it does not remove it
    // from Java indexing.
    "files.exclude": {
        "**/target": true,
        "**/node_modules": true
    }
}
