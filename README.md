# Dropbox MCP Server

[![smithery badge](https://smithery.ai/badge/@amgadabdelhafez/Dropbox-MCP-Server)](https://smithery.ai/server/@amgadabdelhafez/Dropbox-MCP-Server)

This is a Model Context Protocol (MCP) server that integrates with Dropbox. It exposes a set of Dropbox operations as MCP tools, allowing MCP-compatible clients to interact with Dropbox.

## Tools

The following tools are available:

### list_files

Lists files and folders in a Dropbox directory.

- **Input**:
  - `path` (string, optional): Path to the folder (defaults to root "")
- **Output**: JSON array of file and folder entries with metadata

### upload_file

Upload a file to Dropbox.

- **Input**:
  - `path` (string, required): Path where the file should be uploaded (e.g., "/folder/file.txt")
  - `content` (string, required): Base64-encoded file content
- **Output**: Confirmation message with the file path

### download_file

Download a file from Dropbox to local disk and return the file path.

- **Input**:
  - `path` (string, required): Path to the file to download from Dropbox
- **Output**: Path to the downloaded file in the system's temp directory
- **Note**: Files are downloaded to `/tmp/dropbox-mcp-downloads/`. The caller is responsible for reading and cleaning up the downloaded file.

### delete_item

Delete a file or folder from Dropbox.

- **Input**:
  - `path` (string, required): Path to the file or folder to delete
- **Output**: Confirmation message with the deleted path

### create_folder

Create a new folder in Dropbox.

- **Input**:
  - `path` (string, required): Path where the folder should be created (e.g., "/New Folder")
- **Output**: Confirmation message with the created folder path

### copy_item

Copy a file or folder to a new location.

- **Input**:
  - `from_path` (string, required): Path of the source file or folder
  - `to_path` (string, required): Path for the destination file or folder
- **Output**: Confirmation message with both paths

### move_item

Move or rename a file or folder.

- **Input**:
  - `from_path` (string, required): Path of the source file or folder
  - `to_path` (string, required): New path for the file or folder
- **Output**: Confirmation message with both paths

### get_file_metadata

Get metadata for a file or folder.

- **Input**:
  - `path` (string, required): Path to the file or folder
- **Output**: JSON object with file/folder metadata

### search_file_db

Search for files and folders in Dropbox.

- **Input**:
  - `query` (string, required): Search query string
  - `path` (string, optional): Path to search within (defaults to root)
  - `max_results` (number, optional): Maximum number of results to return (1-1000, default: 20)
- **Output**: JSON array of matching files and folders with metadata and highlight information

### get_sharing_link

Create or retrieve a shared link for a file or folder.

- **Input**:
  - `path` (string, required): Path to the file or folder to share
  - `settings` (object, optional): Sharing settings
    - `requested_visibility` (object): Visibility settings (e.g., `{ ".tag": "public" }`)
    - `audience` (object): Audience settings (e.g., `{ ".tag": "public" }`)
    - `access` (object): Access level settings (e.g., `{ ".tag": "viewer" }`)
- **Output**: JSON object with the shared URL and sharing information
- **Note**: If a sharing link already exists, you may need to delete and recreate the file to generate a new link

### get_account_info

Get information about the connected Dropbox account.

- **Input**: None
- **Output**: JSON object with account information:
  ```json
  {
    "account_id": "dbid:...",
    "name": {
      "given_name": "...",
      "surname": "...",
      "familiar_name": "...",
      "display_name": "...",
      "abbreviated_name": "..."
    },
    "email": "...",
    "email_verified": true,
    "country": "...",
    "locale": "...",
    "team": null,
    "account_type": "..."
  }
  ```

### update_access_token

Update the Dropbox access token at runtime. This is particularly useful when:

- The token expires and needs to be refreshed
- You want to switch between different Dropbox accounts
- You need to update permissions without restarting the server

- **Input**:
  - `token` (string, required): New Dropbox access token
- **Output**: Confirmation message
- **Note**: The token must have the necessary permission scopes for the operations you plan to use

## Authentication

The server supports two methods of authentication:

1. **Environment Variable**: Set `DROPBOX_ACCESS_TOKEN` in your environment
2. **Runtime Update**: Use the `update_access_token` tool to set or update the token

The server will automatically handle token validation and provide clear error messages if authentication fails. If a token expires during operation, the server will:

1. Detect the authentication failure
2. Attempt to refresh the token if possible
3. Retry the failed operation automatically

### Token Management Best Practices

1. **Token Storage**:

   - Store the token in a secure location
   - Use environment variables or secure configuration management
   - Never commit tokens to version control

2. **Token Refresh**:

   - Monitor token expiration
   - Implement proper token refresh mechanisms
   - Use the `update_access_token` tool to update tokens when needed

3. **Error Handling**:
   - Watch for authentication errors in responses
   - Implement proper retry mechanisms with exponential backoff
   - Log authentication failures for monitoring

## Configuration

### Environment Variables

- `DROPBOX_ACCESS_TOKEN` (optional): A valid Dropbox access token
  - If not provided via environment, use the `update_access_token` tool

### MCP Settings Configuration

Add the following to your MCP settings file:

```json
{
  "mcpServers": {
    "dropbox": {
      "command": "node",
      "args": ["/path/to/dropbox-mcp-server/build/index.js"],
      "env": {
        "DROPBOX_ACCESS_TOKEN": "your-token-here" // Optional
      }
    }
  }
}
```

## Usage

1. Install the server by adding the MCP server configuration to your settings file
2. Provide authentication either through:
   - Setting the `DROPBOX_ACCESS_TOKEN` environment variable, or
   - Using the `update_access_token` tool after starting the server
3. Start using the tools:

   ```typescript
   // Example: List files in root directory
   await mcp.useTool("dropbox-mcp-server", "list_files", { path: "" });

   // Example: Upload a file
   await mcp.useTool("dropbox-mcp-server", "upload_file", {
     path: "/test.txt",
     content: Buffer.from("Hello World").toString("base64"),
   });

   // Example: Download and read a file
   const result = await mcp.useTool("dropbox-mcp-server", "download_file", {
     path: "/test.txt",
   });
   const filePath = result.content[0].text;
   const fileContent = fs.readFileSync(filePath, "utf8");
   // ... use the file content ...
   fs.unlinkSync(filePath); // Clean up the downloaded file
   ```

## Testing

The server includes a comprehensive test suite that verifies all Dropbox operations. The test suite:

1. Tests all available tools
2. Provides detailed output for each operation
3. Handles error cases gracefully
4. Cleans up after itself (except for the test folder)

### Running the Tests

To run the tests:

1. Make sure you have a valid Dropbox access token in a file named `token` in the root directory
2. Install the required dependencies:
   ```bash
   npm install
   ```
3. Run the test suite using the provided script:

   ```bash
   ./run-tests.sh
   ```

   This script sets the Dropbox access token as an environment variable before running the tests, which helps avoid authentication issues.

   Alternatively, you can run the tests directly:

   ```bash
   npm test
   ```

The test will create a folder named "MCP Test Folder" in your Dropbox root and perform various operations within it. The test file will remain in your Dropbox after the test completes.

### Test Coverage

The test suite covers all major operations and has been verified to pass successfully:

- ✅ Authentication and access token management
- ✅ Account information retrieval
- ✅ File and folder listing
- ✅ Folder creation
- ✅ File upload and download
- ✅ File metadata retrieval
- ✅ File sharing link creation (handles existing links)
- ✅ File searching
- ✅ File copying and moving
- ✅ File deletion

All tests have been verified to pass with proper cleanup and error handling.

### Custom Testing

You can modify the test configuration in `tests/dropbox-operations.test.js` to test with different file names, content, or paths.

### Required Permissions

The Dropbox MCP server requires specific permission scopes to be enabled for your Dropbox access token. Different operations require different permission scopes:

| Operation             | Required Permission Scope                   |
| --------------------- | ------------------------------------------- |
| List files            | `files.metadata.read`                       |
| Upload/download files | `files.content.write`, `files.content.read` |
| Create folders        | `files.content.write`                       |
| Get file metadata     | `files.metadata.read`                       |
| Copy/move files       | `files.content.write`                       |
| Delete files          | `files.content.write`                       |
| Search files          | `files.metadata.read`                       |
| Create sharing links  | `sharing.write`                             |
| Get account info      | `account_info.read`                         |

When generating your access token from the Dropbox App Console, make sure to enable all the permission scopes needed for the operations you plan to use.

### Known Limitations

The test script has the following known limitations:

1. **Authentication Persistence**: Each MCP operation spawns a new server process, and the access token is not persisted between operations. The test script attempts to handle this by detecting authentication errors and re-authenticating, but this may not always work perfectly.

2. **Sharing Link Creation**:
   - Creating sharing links requires the `sharing.write` permission scope in the Dropbox API
   - If a sharing link already exists for a file, the API returns a "shared_link_already_exists" response, which is handled gracefully
   - The test considers both new link creation and "already exists" responses as successful outcomes
   - You can enable the sharing permission in the Dropbox App Console under the "Permissions" tab

### Troubleshooting Tests

If you encounter issues with the tests:

1. **Check your access token**: Make sure your token in the `token` file is valid and has the necessary permissions.

2. **Run tests individually**: You can modify the test script to run only specific operations by commenting out the others.

3. **Increase verbosity**: The test script logs detailed responses for debugging. Look for error messages in these responses.

4. **Check Dropbox API status**: If you're experiencing persistent issues, check if the Dropbox API is experiencing any outages.

## Error Handling

The server provides detailed error messages for common issues:

- Authentication failures (401 errors)
- Invalid paths or parameters (400 errors)
- Missing or expired tokens
- API rate limits or quotas
- Network connectivity issues

## Troubleshooting

If you encounter issues:

1. Check authentication:
   - Verify the access token is valid and not expired
   - Use `update_access_token` to set a new token if needed
2. Verify paths:
   - Paths should be properly formatted (e.g., "/folder/file.txt")
   - Root directory is represented by an empty string ""
3. Check file operations:
   - Ensure you have the necessary permissions
   - Verify base64 encoding/decoding for file content
4. Monitor server logs:
   - The server provides detailed error messages
   - Check response data for API-specific error details

## Development

Built with:

- TypeScript
- Model Context Protocol SDK
- Dropbox SDK v10.34.0
- Dropbox API v2

### Code Organization

The server leverages the official Dropbox SDK for all API operations:

- Type-safe API calls with built-in TypeScript support
- Automatic request/response handling and serialization
- Built-in retry logic and error handling
- Simplified authentication management

The codebase follows clean code principles:

- Consistent use of Dropbox SDK methods
- Standardized error handling with proper error codes
- Clear separation of concerns between modules
- Type-safe operations with TypeScript interfaces

### Implementation Details

The server uses the Dropbox SDK to handle all API operations, which provides several benefits:

1. **Type Safety**:

   - Full TypeScript support for all API methods
   - Compile-time checking of API parameters
   - Automatic type inference for API responses

2. **Error Handling**:

   - Standardized error responses
   - Built-in handling of rate limits and retries
   - Proper error categorization (auth, permissions, etc.)

3. **Authentication**:

   - Automatic token management
   - Built-in token refresh handling
   - Proper scope validation

4. **Performance**:

   - Optimized request handling
   - Connection pooling
   - Automatic request retries

5. **File Handling**:
   - Binary files saved to system temp directory
   - Automatic cleanup handled by OS
   - Safe concurrent downloads with unique filenames
   - Cross-platform compatibility using os.tmpdir()

## License
