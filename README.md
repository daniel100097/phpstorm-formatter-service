# PhpStorm Formatter as a Service

## Usage
To test the service, execute the following commands:

```bash
curl --request POST \
  --url http://localhost:8084/aa/file.php \
  --data '<?php echo     111;'
```

```bash
curl --request POST \
  --url http://localhost:8084/aa/file.js \
  --data 'let a = 1;a=2'
```

## Installation
To set up this service, you need:

- PhpStorm
- The "Live Plugins" plugin for PhpStorm
- The Groovy script provided below
- A folder named formatter at the root of your project (refer to the script for details)

Please note that the development of this tool was done in PhpStorm without syntax highlighting or formatting, and with limited Java knowledge.


```groovy
import com.sun.net.httpserver.HttpServer
import com.sun.net.httpserver.HttpHandler
import com.sun.net.httpserver.HttpExchange
import com.intellij.psi.codeStyle.CodeStyleManager
import com.intellij.openapi.vfs.LocalFileSystem;
import com.intellij.openapi.vfs.VirtualFile;
import com.intellij.openapi.fileEditor.FileDocumentManager;
import com.intellij.openapi.fileEditor.FileDocumentManager
import com.intellij.openapi.project.Project;
import com.intellij.psi.PsiFile;
import com.intellij.psi.PsiManager;
import com.intellij.openapi.command.WriteCommandAction;
import com.intellij.openapi.editor.Document;
import java.nio.charset.StandardCharsets;
import com.intellij.psi.PsiDocumentManager


class FormatterHandler implements HttpHandler {
    private Project project
    private File logFile

    FormatterHandler(Project project) {
        this.project = project
        this.logFile = new File(project.getBasePath(), "log.txt") // Assuming getBasePath() returns the root project directory
    }

    private void log(String message) {
        FileWriter writer = new FileWriter(logFile, true) // true to append, false to overwrite
        writer.write(message + "\n")
        writer.close()
    }

    private VirtualFile getParentDirectory() {
        VirtualFile parentDirectory = LocalFileSystem.getInstance().findFileByPath(project.getBasePath() + "/formatter");

        // Ensure the formatter directory exists
        if (parentDirectory == null || !parentDirectory.isDirectory()) {
            throw new RuntimeException("Formatter directory does not exist");
        }
        return parentDirectory;
    }

    private VirtualFile getVirtualFile(String fileName) {
        VirtualFile parentDirectory = getParentDirectory();

        VirtualFile virtualFile = LocalFileSystem.getInstance().findFileByPath(project.getBasePath() + "/" + fileName);
        if (virtualFile == null) {
            virtualFile = parentDirectory.createChildData(this, fileName);
        }
        return virtualFile;
    }
    private Document getDocument(VirtualFile virtualFile) {
        Document document = FileDocumentManager.getInstance().getDocument(virtualFile);
        if (document == null) {
            throw new RuntimeException("Document is null");
        }
        return document;
    }

    private PsiFile getPsiFile(VirtualFile virtualFile) {
        PsiFile psiFile = PsiManager.getInstance(project).findFile(virtualFile);
        if (psiFile == null) {
            throw new RuntimeException("PsiFile is null");
        }
        return psiFile;
    }

    @Override
    void handle(HttpExchange exchange) throws IOException {
        try{
            // VirtualFileManager.getInstance().syncRefresh()

            if (!"POST".equals(exchange.getRequestMethod())) {
                exchange.sendResponseHeaders(405, -1) // Method Not Allowed
                return
            }
            String path = exchange.getRequestURI().getPath()
            String extension = path.lastIndexOf(".") > 0 ? path.substring(path.lastIndexOf(".")) : ".txt"

            String fileName = UUID.randomUUID().toString() + extension
//             File file = new File(project.getBasePath(), fileName)
            String formattedContent = null;
            VirtualFile virtualFile = null;
            Document document = null;

            try{
                WriteCommandAction.runWriteCommandAction(project, () -> {
                    virtualFile = getVirtualFile(fileName);
                    document = getDocument(virtualFile);
                    InputStream requestBody = exchange.getRequestBody();
                    String content = new String(requestBody.readAllBytes(), StandardCharsets.UTF_8);

                    document.setText(content);

                    FileDocumentManager.getInstance().saveDocument(document);

                    FileDocumentManager fileDocumentManager = FileDocumentManager.getInstance();
                    fileDocumentManager.saveDocument(document);

                    PsiDocumentManager.getInstance(project).commitAllDocuments();
                    PsiFile psiFile = getPsiFile(virtualFile);



                    CodeStyleManager.getInstance(project).reformat(psiFile);
                    PsiDocumentManager.getInstance(project).doPostponedOperationsAndUnblockDocument(document);
                    PsiDocumentManager.getInstance(project).commitDocument(document);
                    // PsiDocumentManager.getInstance(project).commitAllDocuments();


                    // VirtualFile parentDirectory = getParentDirectory();
                    // parentDirectory.refresh(false, true);
                    formattedContent = document.getText()
                    virtualFile.delete(this);
                });
                if (formattedContent == null) {
                    exchange.sendResponseHeaders(500, -1) // Internal Server Error
                    return
                }

                exchange.sendResponseHeaders(200, formattedContent.getBytes().length)
                OutputStream os = exchange.getResponseBody()
                os.write(formattedContent.getBytes())
                os.close()
            }catch (Exception e) {
                log("Error: " + e.getMessage())
                exchange.sendResponseHeaders(500, -1) // Internal Server Error
                log("Error: " + e.getMessage() + " at line " + e.getStackTrace()[0].getLineNumber())
            }
            // file.delete()
            // LocalFileSystem.getInstance().refreshAndFindFileByPath(file.getAbsolutePath())
        } catch (Exception e) {
            log("Error: " + e.getMessage())
            // log the line number and the error message
            log("Error: " + e.getMessage() + " at line " + e.getStackTrace()[0].getLineNumber())
            exchange.sendResponseHeaders(500, -1) // Internal Server Error
        }
    }
}

int port = 8084
HttpServer server = HttpServer.create(new InetSocketAddress(port), 0)
// start a new thread to keep the server running
server.createContext("/", new FormatterHandler(project))
server.setExecutor(null) // creates a default executor
server.start()

````
