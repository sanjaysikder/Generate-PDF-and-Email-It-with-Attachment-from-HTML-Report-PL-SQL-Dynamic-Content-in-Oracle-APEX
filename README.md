# 📄 Generate-PDF-and-Email-It-with-Attachment-from-HTML-Report-PL-SQL-Dynamic-Content-in-Oracle-APEX

This guide helps you integrate a **"Generate PDF and Email it with Attachment"** feature in your Oracle APEX application using html2pdf.js.

---

## 📁 1. Upload the JavaScript Library

1. Download the html2pdf.bundle.min.js library:
   👉 https://ekoopmans.github.io/html2pdf.js/

3. In Oracle APEX:
   - Go to **Shared Components** → **Static Application Files**
   - Click **Upload File**
   - Upload the file html2pdf.bundle.min.js
   - #APP_FILES#html2pdf.bundle.min.js

---
## 🧠 2. Add This Static File URL to Page HTML Header

 - Add This Static File to Page HTML Header.
```HTML Header
<script src="#APP_FILES#html2pdf.bundle.min.js"></script>
```

## 🧠 3. Create ajax callback Process give it a name (STORE_PDF)

```pl/sql code

DECLARE
  l_collection_name VARCHAR2(100) := 'GENERATE_PDF';
  l_blob            BLOB;
  l_filename        VARCHAR2(100);
  l_mime_type       VARCHAR2(100);
  l_token           VARCHAR2(32000);
  email_id number;
  l_body_html VARCHAR2(32767);
begin
BEGIN
  -- Get mime type from x01 or default to 'image/png'
  l_mime_type := NVL(apex_application.g_x02, 'application/pdf');
  -- Create filename based on the current date and time
  l_filename := 'Invoice-' ||
              TO_CHAR(SYSDATE, 'DD-MM-YYYY') || '_' ||
              LOWER(TO_CHAR(SYSDATE, 'HH.MIPM'));
  -- Append appropriate file extension based on mime type
  IF l_mime_type = 'image/png' THEN
    l_filename := l_filename || '.png';
  ELSIF l_mime_type = 'image/jpeg' THEN
    l_filename := l_filename || '.jpg';
  ELSIF l_mime_type = 'application/pdf' THEN
    l_filename := l_filename || '.pdf';
  END IF;
  
  -- Create a temporary BLOB to store the decoded image data
  DBMS_LOB.createtemporary(l_blob, FALSE, DBMS_LOB.SESSION);
 
  -- Decode the base64 strings in the f01 array and append them to the BLOB
  FOR i IN 1 .. apex_application.g_f01.COUNT LOOP
    l_token := wwv_flow.g_f01(i);
    IF LENGTH(l_token) > 0 THEN
      DBMS_LOB.APPEND(l_blob, TO_BLOB(UTL_ENCODE.base64_decode(UTL_RAW.cast_to_raw(l_token))));
    END IF;
  END LOOP;
 
  -- Check if the collection exists; if not, create it
  IF NOT apex_collection.collection_exists(p_collection_name => l_collection_name) THEN
    apex_collection.create_collection(l_collection_name);
  END IF;
  
  -- Add the BLOB as a member to the collection, if it is not null
  IF DBMS_LOB.getlength(l_blob) IS NOT NULL THEN
    apex_collection.add_member(
      p_collection_name => l_collection_name,
      p_c001            => l_filename,   -- filename
      p_c002            => l_mime_type,  -- mime type
      p_d001            => SYSDATE,      -- date created
      p_blob001         => l_blob        -- BLOB image content
    );
  END IF;

  -- Free temporary BLOB
  DBMS_LOB.freetemporary(l_blob);

    -- Return success status
    apex_json.open_object;
    apex_json.write('success', true);
    apex_json.close_object;

    EXCEPTION
    WHEN OTHERS THEN
        -- Free temporary BLOB in case of an error
        --IF DBMS_LOB.IS_TEMPORARY(l_blob) = 1 THEN
            --DBMS_LOB.FREETEMPORARY(l_blob);
       -- END IF;
        -- Return failure status
        apex_json.open_object;
        apex_json.write('success', false);
        apex_json.close_object;
        RAISE;
     apex_application.g_print_success_message := '<span>The email has been sent successfully to  '||:P84_EMAIL||'</span>';
--apex_collection.truncate_collection('GENERATE_PDF') ;
 end;
END;

```


## 🧠 4. Add JavaScript to Page-Level Global Variable Function

1. Go to the page where you want the export feature.
2. Under **Page Attributes**, scroll to **Function and Global Variable Declaration**.
3. Paste the following code:

```javascript

//Generate PDF with Download and send email

function generatePDFWithSendEmail() {
    var element = document.getElementById('printID'); // Change to your container ID
    
// Generate timestamp-based filename
    var currentDate = new Date();
    var day = ('0' + currentDate.getDate()).slice(-2);
    var month = ('0' + (currentDate.getMonth() + 1)).slice(-2);
    var year = currentDate.getFullYear();
    var hours = currentDate.getHours();
    var minutes = ('0' + currentDate.getMinutes()).slice(-2);
    var ampm = hours >= 12 ? 'PM' : 'AM';
    hours = hours % 12;
    hours = hours ? hours : 12; // Handle hour '0' as '12'
    hours = ('0' + hours).slice(-2);

    // Human-readable display format (optional use)
    var displayDate = day + '/' + month + '/' + year + ' ' + hours + ':' + minutes + ampm;

    // Safe filename format (slashes and colons replaced)
    var filename = 'Invoice-' + day + '-' + month + '-' + year + '_' + hours + '-' + minutes + ampm + '.pdf';

    var opt = {
        margin: 0.5,
        filename: filename,
        image: { type: 'jpeg', quality: 0.98 },
        html2canvas: { scale: 2 },
        jsPDF: { unit: 'in', format: 'a4', orientation: 'portrait' }
    };

    html2pdf().set(opt).from(element).outputPdf('blob').then(function(pdfBlob) {
        var reader = new FileReader();
        reader.onloadend = function() {
            var base64data = reader.result.split(',')[1];
            apex.server.process('STORE_PDF', {
                x01: filename, // Filename
                x02: 'application/pdf', // MIME type
                x03: new Date().toISOString(), // Creation date
                f01: base64data // PDF Blob
            }, {
                dataType: 'text',
                success: function(data) {
                    alert('PDF Generated successfully!');
                       //download pdf 
                        html2pdf().set(opt).from(element).save();
                        //end download
                },
                error: function(xhr, status, error) {
                    console.error('Error storing PDF:', error);
                }
            });
        };
        reader.readAsDataURL(pdfBlob);
    });
}

```

## 4. 📥 Download PDF and Email It with Attachment via Dynamic Action (Click Button)

### ✅ Step-by-Step:

1. **Create a Button**
   - Label it (e.g., `Download and Send Email`)
   - Position it on the same APEX page as your report (HTML container with `printID`).

2. **Create a Dynamic Action**
   - **Event**: `Click`
   - **Selection Type**: `Button`
   - **Button**: Select the button you just created (e.g., `Download and Send Email`)

3. **Add a True Action**
   - **Action Type**: `Execute JavaScript Code`
   - **Code**:
     ```javascript
      generatePDFWithSendEmail();

     ```

4. **Add Another True Action**
   - **Action Type**: `Execute server-side Code`
   - **Code**:
     
 ```pl/sql code
   DECLARE
    email_id    NUMBER;
    l_body_html VARCHAR2(32767);
   BEGIN
    -- Email HTML Body
    l_body_html := 
        '<html>
           <body>
             <h1>Test Email</h1>
             <p>This is a static email sent from APEX.</p>
           </body>
         </html>';
    -- Send Email
    email_id := APEX_MAIL.SEND(
                    p_to        => 'sanjaysikder71@gmail.com',
                    p_from      => 'sanjay@akijinsaf.com',
                    p_body      => l_body_html,
                    p_body_html => l_body_html,
                    p_subj      => 'Hello Test'
                );
    -- Attach files from the collection to the email
    FOR file_rec IN (
        SELECT 
            c001    AS filename,
            c002    AS mime_type, 
            d001    AS create_date, 
            blob001 AS blob_file 
        FROM APEX_COLLECTIONS
        WHERE COLLECTION_NAME = 'GENERATE_PDF'
    ) LOOP
        APEX_MAIL.ADD_ATTACHMENT(
            p_mail_id    => email_id,
            p_attachment => file_rec.blob_file,
            p_filename   => file_rec.filename,
            p_mime_type  => file_rec.mime_type
        );
    END LOOP;

    -- Commit to send the email
    APEX_MAIL.PUSH_QUEUE();

    -- Show success message
    apex_application.g_print_success_message := 
        '<span>The email has been sent successfully to ' || :P84_EMAIL || '</span>';

    -- Clear the collection
    APEX_COLLECTION.TRUNCATE_COLLECTION('GENERATE_PDF');
END;

``
### 💡 Notes:
- Ensure the element you want to print has the ID `printID`.
- `html2pdf.js` must be loaded on the page (see section 2 of the documentation for CDN or static file setup).
- The `generatePDFWithSendEmail` function must be declared globally (in Page > JavaScript > Function and Global Variable Declaration).
- Create a Dynamic Action for the button’s click event to call the JavaScript function `Execute JavaScript Code`.
- Add Another True Action `Execute server-side Code`.




# Thank You
## Sanjay Sikder
