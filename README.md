![Banner image](https://user-images.githubusercontent.com/10284570/173569848-c624317f-42b1-45a6-ab09-f0ea3c247648.png)

[WIP!]

# n8n-node-yandex-ocr

This repo contains n8n node to proceed files at Yandex OCR

# Yandex OCR (Document Recognition) – n8n Node  

**Node type:** `yandexOcr`  
**Category:** *Cloud Services → Yandex Cloud*  

## Overview  

The **Yandex OCR** node sends binary documents (images, PDFs, TIFFs, etc.) to the Yandex Cloud Vision API’s OCR service and returns the extracted text along with detailed metadata. It is a fully‑featured wrapper that handles authentication, request construction, error handling, and result parsing, allowing you to integrate optical character recognition into any n8n workflow with just a few clicks.

## Key Features  

| Feature | Description |
|---------|-------------|
| **Supports all Yandex OCR input formats** – JPEG, PNG, BMP, TIFF, PDF (up to 20 MB per request). |
| **Automatic credential handling** – uses the built‑in *Yandex Cloud OAuth2* credential type. |
| **Language selection** – specify one or more ISO‑639‑1 language codes to improve recognition accuracy. |
| **Folder ID configuration** – work with multiple Yandex Cloud folders without changing the node code. |
| **Custom OCR options** – enable/disable `detectOrientation`, `detectLanguage`, `extractTextFromTables`, etc. |
| **Binary output** – returns the raw OCR JSON response as a JSON object; you can also output plain text only. |
| **Error handling** – returns a clear error object with HTTP status, request ID and Yandex error message. |

## Credentials  

The node requires a **Yandex Cloud OAuth2** credential:

| Field | Description |
|-------|-------------|
| **OAuth Token** | A valid IAM token (`IAM token`) generated from a service account or OAuth flow. |
| **Folder ID** | The Yandex Cloud folder where the OCR service is enabled. (Can also be overridden per‑node.) |

Create the credential in **Settings → Credentials → New Credential → Yandex Cloud OAuth2** and supply the token and default folder ID.

## Node Properties  

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| **File** | *Binary* | ✅ | The binary data of the document to be processed. Connect a *Read Binary File* node or any node that outputs binary data. |
| **File Name** | *String* | ✅ | The original file name (including extension). Used by Yandex to infer MIME type. |
| **Language(s)** | *String (comma‑separated)* | ❌ | ISO‑639‑1 language codes (e.g., `en,ru`). If omitted, Yandex attempts auto‑detection. |
| **Folder ID** | *String* | ❌ | Overrides the folder ID from the credential. |
| **Detect Orientation** | *Boolean* | ❌ (default `true`) | When `true`, Yandex will try to correct image rotation. |
| **Detect Language** | *Boolean* | ❌ (default `false`) | When `true`, Yandex will return the detected language for each block. |
| **Extract Text From Tables** | *Boolean* | ❌ (default `false`) | Enables table structure detection (requires PDF input). |
| **Return Plain Text Only** | *Boolean* | ❌ (default `false`) | If `true`, the node will output only the concatenated plain text string instead of the full JSON response. |
| **Additional Request Headers** | *Key‑Value* | ❌ | Any extra HTTP headers you need to send (e.g., `User-Agent`). |
| **Timeout (ms)** | *Number* | ❌ (default `60000`) | Maximum time to wait for the OCR request before aborting. |

### Advanced – JSON Parameters (optional)  

If you need to send a custom request body, you can enable **“Raw JSON Parameters”** and provide a JSON object that will be merged with the default payload. This is useful for future Yandex OCR features not yet exposed via the UI.

## Input Data  

The node expects **binary data** on the `binary` property of the incoming item. The binary property name can be changed in the node settings (default is `data`). Example input from a *Read Binary File* node:

```json
{
  "binary": {
    "data": {
      "data": "<Buffer ...>",
      "mimeType": "application/pdf",
      "fileName": "invoice.pdf"
    }
  }
}
```

If the incoming item does not contain binary data, the node will throw a clear validation error.

## Output Data  

The node returns one item per input item (preserving order). The output contains a JSON field `ocrResult` and, optionally, a plain‑text field `text`.

### Full JSON response (default)

```json
{
  "json": {
    "ocrResult": {
      "requestId": "a1b2c3d4-5678-90ab-cdef-1234567890ab",
      "languageDetected": "en",
      "pages": [
        {
          "pageNumber": 1,
          "blocks": [
            {
              "text": "Invoice #12345",
              "boundingBox": {...},
              "confidence": 0.99
            },
            …
          ]
        }
      ],
      "fullText": "Invoice #12345\nDate: 2025‑09‑30\n..."
    }
  },
  "binary": {}
}
```

### Plain‑text only (if **Return Plain Text Only** = `true`)

```json
{
  "json": {
    "text": "Invoice #12345\nDate: 2025‑09‑30\n..."
  },
  "binary": {}
}
```

If the request fails, the node outputs a JSON object with the fields `error`, `statusCode`, `requestId`, and `message`.

## Usage Example  

```mermaid
flowchart TD
    A[Start] --> B[Read Binary File (invoice.pdf)]
    B --> C[Yandex OCR]
    C --> D[Set Variable (ocrText)]
    D --> E[Send Email with OCR result]
    E --> F[End]
```

**Step‑by‑step**

1. **Read Binary File** – loads `invoice.pdf` from the local filesystem (or from an HTTP request).  
2. **Yandex OCR** – configure:
   * *File*: `data` (binary property from the previous node)  
   * *Language(s)*: `en,ru`  
   * *Return Plain Text Only*: `true`  
3. **Set Variable** – stores `{{$json["text"]}}` in a workflow variable for later use.  
4. **Send Email** – inject the variable into the email body.

## Known Limitations  

| Limitation | Work‑around |
|------------|-------------|
| **Maximum file size 20 MB** (Yandex limit) | Split large PDFs into pages before sending, or compress images. |
| **Only one file per request** | Use an *Iterator* or *SplitInBatches* node to process multiple files sequentially. |
| **No built‑in pagination for large PDFs** | Process each page separately by extracting pages with a PDF‑split node, then feed each page to the OCR node. |
| **IAM token expiration** – tokens are valid for 12 h. | Use the *Yandex Cloud OAuth2* credential which automatically refreshes the token when needed. |

## Implementation Details (for developers)

- **Endpoint**: `POST https://vision.api.cloud.yandex.net/vision/v1/ocr:recognize`
- **Headers**:  
  - `Authorization: Bearer <IAM token>`  
  - `Content-Type: multipart/form-data`
- **Body**: `multipart/form-data` with fields:
  - `folderId` (string)  
  - `document` (binary file)  
  - `lang` (comma‑separated list) – optional  
  - Additional flags (`detectOrientation`, `detectLanguage`, `extractTextFromTables`) as JSON booleans.
- **Response parsing** is performed with `JSON.parse` and the relevant data is mapped to the output fields described above.
- **Error handling**: non‑2xx responses are caught, parsed, and emitted as a structured error object (see *Output Data*).

---

**Ready to use** – just drag the **Yandex OCR** node into your workflow, connect a binary source, configure the language(s) you need, and start extracting text with Yandex Cloud’s powerful OCR engine!

## License

[MIT](https://github.com/n8n-io/n8n-nodes-starter/blob/master/LICENSE.md)
