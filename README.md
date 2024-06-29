# fakewebserver
Non-admin Windows PowerShell command to launch a fake web server for local dev purpose (no Cross-Origin error). Close Powershell window to stop the fake web server (or Ctrl+C and make at least one last request).

The fake web server listen on `http://localhost:$port/`.

[Script version with parameters.](#powershell-script-version)

## Settings
- `$root` : local directory folder, acting as server root. (default value : ".")
- `$port` : port number. (default value : 8000)
- `$index` : file to serve on "/". (default value : "index.html")

## Command
```PowerShell
$root=".";$port=8000;$index="index.html";$listener=New-Object System.Net.HttpListener;$listener.Prefixes.Add("http://localhost:$port/");$listener.Start();Write-Host "Listening at http://localhost:$port/" -BackgroundColor Green;function HandleRequest{param([System.Net.HttpListenerContext]$context);$request=$context.Request;$response=$context.Response;$path=$request.Url.LocalPath.Substring(1);$time=$(Get-Date).ToString("yyyy-MM-dd HH:mm:ss");if([string]::IsNullOrEmpty($path)-or(Test-Path -Path (Join-Path -Path $root -ChildPath $path) -PathType Container)){$path=$index};$fullPath=Join-Path -Path $root -ChildPath $path;$summary="$time | URL: $($request.Url.LocalPath) | File: $fullPath";if(Test-Path -Path $fullPath -PathType Leaf){try{$bytes=[System.IO.File]::ReadAllBytes($fullPath);$response.ContentType=Get-ContentType $fullPath;$response.ContentLength64=$bytes.Length;$response.OutputStream.Write($bytes,0,$bytes.Length);Write-Host "$summary | Status: OK" -ForegroundColor Green}catch{$response.StatusCode=500;$response.StatusDescription="Internal Server Error";Write-Host "$summary | Status: ERROR" -ForegroundColor Red}}else{$response.StatusCode=404;$response.StatusDescription="Not Found";$errorHtml="<html><head><title>404 Not Found</title></head><body><h1>404 - File Not Found</h1><p>The requested URL $($request.Url.LocalPath) was not found on this server.</p></body></html>";$errorBytes=[System.Text.Encoding]::UTF8.GetBytes($errorHtml);$response.ContentType="text/html";$response.ContentLength64=$errorBytes.Length;$response.OutputStream.Write($errorBytes,0,$errorBytes.Length);Write-Host "$summary | Status: 404 Not Found" -ForegroundColor Yellow};$response.OutputStream.Close()};function Get-ContentType{param([string]$path);switch([System.IO.Path]::GetExtension($path).ToLower()){".html"{"text/html"};".htm"{"text/html"};".txt"{"text/plain"};".css"{"text/css"};".js"{"application/javascript"};".jpg"{"image/jpeg"};".jpeg"{"image/jpeg"};".png"{"image/png"};".gif"{"image/gif"};".svg"{"image/svg+xml"};".pdf"{"application/pdf"};".json"{"application/json"};".xml"{"application/xml"};default{"application/octet-stream"}}};try{while($listener.IsListening){$context=$listener.GetContext();HandleRequest $context}}finally{$listener.Stop()}
```

## Exploded code with comments
```PowerShell
# Set the root directory where files are served from
$root = "."

# Set the port number
$port = 8000

# Set the file to serve on the base URL
$index = "index.html"

# Create a new HTTP listener
$listener = New-Object System.Net.HttpListener

# Add the prefix URL to listen on
$listener.Prefixes.Add("http://localhost:$port/")

# Start listening for incoming HTTP requests
$listener.Start()

# Output a message indicating the server is listening
Write-Host "Listening at http://localhost:$port/" -BackgroundColor Green

# Define a function to handle incoming HTTP requests
function HandleRequest {
    param([System.Net.HttpListenerContext]$context)

    $request = $context.Request
    $response = $context.Response

    # Extract the path from the request URL
    $path = $request.Url.LocalPath.Substring(1)

    # Define when the request was made
    $time=$(Get-Date).ToString("yyyy-MM-dd HH:mm:ss")

    # If the path is empty or it points to a directory, default to serving 'index.html'
    if ([string]::IsNullOrEmpty($path) -or (Test-Path -Path (Join-Path -Path $root -ChildPath $path) -PathType Container)) {
        $path = $index
    }

    # Construct the full path to the requested file
    $fullPath = Join-Path -Path $root -ChildPath $path

    # Summary of the request to display
    $summary="$time | URL: $($request.Url.LocalPath) | File: $fullPath"

    # Check if the requested file exists
    if (Test-Path -Path $fullPath -PathType Leaf) {
        try {
            # Read the contents of the file as bytes
            $bytes = [System.IO.File]::ReadAllBytes($fullPath)

            # Set the content type of the response based on file extension
            $response.ContentType = Get-ContentType $fullPath

            # Set the content length of the response
            $response.ContentLength64 = $bytes.Length

            # Write the file bytes to the output stream
            $response.OutputStream.Write($bytes, 0, $bytes.Length)

            # Output a message indicating the request was successfull
            Write-Host "$summary | Status: OK" -ForegroundColor Green
        }
        catch {
            # If there's an error reading the file, return a 500 Internal Server Error response
            $response.StatusCode = 500
            $response.StatusDescription = "Internal Server Error"
            
            # Output a message indicating there was an error reading the file
            Write-Host "$summary | Status: ERROR" -ForegroundColor Red
        }
    }
    else {
        # If the file does not exist, return a 404 Not Found response
        $response.StatusCode = 404
        $response.StatusDescription = "Not Found"

        # Prepare HTML content for the error page
        $errorHtml = "<html><head><title>404 Not Found</title></head><body><h1>404 - File Not Found</h1><p>The requested URL $($request.Url.LocalPath) was not found on this server.</p></body></html>"
        $errorBytes = [System.Text.Encoding]::UTF8.GetBytes($errorHtml)

        # Set the content type of the error response
        $response.ContentType = "text/html"

        # Set the content length of the error response
        $response.ContentLength64 = $errorBytes.Length

        # Write the error content to the output stream
        $response.OutputStream.Write($errorBytes, 0, $errorBytes.Length)

        # Output a message indicating the file was not found
        Write-Host "$summary | Status: 404 Not Found" -ForegroundColor Yellow
    }

    # Close the output stream of the response
    $response.OutputStream.Close()
}

# Function to determine the content type based on file extension
function Get-ContentType {
    param([string]$path)

    switch ([System.IO.Path]::GetExtension($path).ToLower()) {
        ".html" { "text/html" }
        ".htm"  { "text/html" }
        ".txt"  { "text/plain" }
        ".css"  { "text/css" }
        ".js"   { "application/javascript" }
        ".jpg"  { "image/jpeg" }
        ".jpeg" { "image/jpeg" }
        ".png"  { "image/png" }
        ".gif"  { "image/gif" }
        ".svg"  { "image/svg+xml" }
        ".pdf"  { "application/pdf" }
        ".json" { "application/json" }
        ".xml"  { "application/xml" }
        default { "application/octet-stream" }
    }
}

try {
    # Continuously handle incoming HTTP requests while the listener is active
    while ($listener.IsListening) {
        $context = $listener.GetContext()
        HandleRequest $context
    }
}
finally {
    # Stop the listener when done
    $listener.Stop()
}
```
## PowerShell script version

```PowerShell
param(
    # Set the root directory where files are served from
    [String]$root = ".",

    # Set the port number
    [UInt16]$port = 8000,

    # Set the file to serve on the base URL
    [String]$index = "index.html"
)

# Create a new HTTP listener
$listener = New-Object System.Net.HttpListener

# Add the prefix URL to listen on
$listener.Prefixes.Add("http://localhost:$port/")

# Start listening for incoming HTTP requests
$listener.Start()

# Output a message indicating the server is listening
Write-Host "Listening at http://localhost:$port/" -BackgroundColor Green

# Define a function to handle incoming HTTP requests
function HandleRequest {
    param([System.Net.HttpListenerContext]$context)

    $request = $context.Request
    $response = $context.Response

    # Extract the path from the request URL
    $path = $request.Url.LocalPath.Substring(1)

    # Define when the request was made
    $time=$(Get-Date).ToString("yyyy-MM-dd HH:mm:ss")

    # If the path is empty or it points to a directory, default to serving 'index.html'
    if ([string]::IsNullOrEmpty($path) -or (Test-Path -Path (Join-Path -Path $root -ChildPath $path) -PathType Container)) {
        $path = $index
    }

    # Construct the full path to the requested file
    $fullPath = Join-Path -Path $root -ChildPath $path

    # Summary of the request to display
    $summary="$time | URL: $($request.Url.LocalPath) | File: $fullPath"

    # Check if the requested file exists
    if (Test-Path -Path $fullPath -PathType Leaf) {
        try {
            # Read the contents of the file as bytes
            $bytes = [System.IO.File]::ReadAllBytes($fullPath)

            # Set the content type of the response based on file extension
            $response.ContentType = Get-ContentType $fullPath

            # Set the content length of the response
            $response.ContentLength64 = $bytes.Length

            # Write the file bytes to the output stream
            $response.OutputStream.Write($bytes, 0, $bytes.Length)

            # Output a message indicating the request was successfull
            Write-Host "$summary | Status: OK" -ForegroundColor Green
        }
        catch {
            # If there's an error reading the file, return a 500 Internal Server Error response
            $response.StatusCode = 500
            $response.StatusDescription = "Internal Server Error"
            
            # Output a message indicating there was an error reading the file
            Write-Host "$summary | Status: ERROR" -ForegroundColor Red
        }
    }
    else {
        # If the file does not exist, return a 404 Not Found response
        $response.StatusCode = 404
        $response.StatusDescription = "Not Found"

        # Prepare HTML content for the error page
        $errorHtml = "<html><head><title>404 Not Found</title></head><body><h1>404 - File Not Found</h1><p>The requested URL $($request.Url.LocalPath) was not found on this server.</p></body></html>"
        $errorBytes = [System.Text.Encoding]::UTF8.GetBytes($errorHtml)

        # Set the content type of the error response
        $response.ContentType = "text/html"

        # Set the content length of the error response
        $response.ContentLength64 = $errorBytes.Length

        # Write the error content to the output stream
        $response.OutputStream.Write($errorBytes, 0, $errorBytes.Length)

        # Output a message indicating the file was not found
        Write-Host "$summary | Status: 404 Not Found" -ForegroundColor Yellow
    }

    # Close the output stream of the response
    $response.OutputStream.Close()
}

# Function to determine the content type based on file extension
function Get-ContentType {
    param([string]$path)

    switch ([System.IO.Path]::GetExtension($path).ToLower()) {
        ".html" { "text/html" }
        ".htm"  { "text/html" }
        ".txt"  { "text/plain" }
        ".css"  { "text/css" }
        ".js"   { "application/javascript" }
        ".jpg"  { "image/jpeg" }
        ".jpeg" { "image/jpeg" }
        ".png"  { "image/png" }
        ".gif"  { "image/gif" }
        ".svg"  { "image/svg+xml" }
        ".pdf"  { "application/pdf" }
        ".json" { "application/json" }
        ".xml"  { "application/xml" }
        default { "application/octet-stream" }
    }
}

try {
    # Continuously handle incoming HTTP requests while the listener is active
    while ($listener.IsListening) {
        $context = $listener.GetContext()
        HandleRequest $context
    }
}
finally {
    # Stop the listener when done
    $listener.Stop()
}
```
