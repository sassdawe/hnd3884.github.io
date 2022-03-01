Vulnerability Analysis CVE-2022-22005

# Microsoft Sharepoint
![image](/assets/posts/cve-2022-22005/1.png)

SharePoint is a platform for sharing and managing content, knowledge, and apps to support teamwork, quickly finding information, and collaborating seamlessly across the organization. More than 200,000 organizations and 190 million people use SharePoint for intranets, team sites, and content management. The number above is enough to see that this is always a big target for security researchers looking for vulnerabilities.

With SharePoint, users can create an intranet (or intranet system) that works like any other website. In addition to a large site for the organization, sharepoint can divide small sub-sites for each group and internal department. Besides, this is a great content sharing management platform with customizable lists as much as you want. Some types of list are built-in on Sharepoint such as list of images, documents, forms... In addition to the built-in lists, users can install a new list and customize the properties of that list as they like. The powerful toolset for customizing on Sharepoint are Sharepoint Designer and InfoPath Designer. 

# CVE-2022-22005
Microsoft's February - 2022 patch fixes a vulnerability with code CVE-2022-22005. This vulnerability allows an attacker to execute code remotely and is scored 8.8 on the CVSSv3 calculator. Affected versions are listed below 
- Microsoft SharePoint Server Subscription Edition
- Microsoft SharePoint Server 2019
- Microsoft SharePoint Enterprise Server 2013 Service Pack 1
- Microsoft SharePoint Enterprise Server 2016

The analysis below the article was performed on Microsoft SharePoint Enterprise Server 2016 version

## Phân tích bản vá
Install patch January and February 2022 of Sharepoint 2016, gather Sharepoint dll files and decompile into source. Add a few post-production steps to remove unnecessary elements (comments, ...). Finally compare the two patches to find where the code is used by microsoft developers to patch. A Deserialization patch location is found at Microsoft.Office.Server.Internal.Charting.UI.WebControls.ChartPreviewImage.loadChartImage()

![image](/assets/posts/cve-2022-22005/2.png)

The patch uses a binder that limits what types are allowed to deserialize, which is what Microsoft used to do for bugs like this in the past. About the deserialize error you can learn more at [here](https://portswigger.net/web-security/deserialization).

## Trace code
Learn a little bit about chart - chart on Sharepoint, this is a webpartpage - component of a page on Sharepoint. So it can be understood that in order for the data to go to the deserialize location, there must be a user account __ with permission to create page__. Combining debugging and creating a page that uses the chart, the code is called when the chart is loading data. Observe the function that causes the loadChartImage error, the deserialize data located in the buffer variable is set to the value via the FetchBinaryData(sessionKey) function.

```csharp
// Microsoft.Office.Server.Internal.Charting.UI.WebControls.ChartPreviewImage.loadChartImage()
private ChartImageSessionBlock loadChartImage()
{
    byte[] buffer = CustomSessionState.FetchBinaryData(this.sessionKey);
    ChartImageSessionBlock result = null;
    using (
        MemoryStream memoryStream = new MemoryStream(buffer)
    )
    {
        IFormatter formatter = new BinaryFormatter();
        result = (ChartImageSessionBlock)formatter.Deserialize(memoryStream);
    }
    return result;
}
```

The code related to session state in Sharepoint. This is a mechanism to store the state of an object in the sharepoint, that state can be a file, an image, ... or specifically in this case a ChartImageSessionBlock object after serializing. These states will be stored in the database as binary data and mapped to a session key. So to exploit this error we need to control the binary data in the database, then through the loadChartImage function to deserialize an arbitrary object. Using the Burp Suite tool during debugging we will get a request to trigger the error function.

```
GET /_layouts/15/Chart/WebUI/Controls/ChartPreviewImage.aspx?sk=5264ebfb259840faa703bdbc976e069b_74929f85360d499d9f1d4f337bf49300&hash=2551012 HTTP/1.1
Host: sharepoint2016:33257
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.82 Safari/537.36
Referer: http://sharepoint2016:33257/SitePages/testpage.aspx
Cookie: stsSyncAppName=Client; stsSyncIconPath=; WSS_FullScreenMode=false
Connection: Keep-Alive
```

The variable sk here is the sessionKey passed into the FetchBinaryData function, of the form guid1_guid2 where guid1 is the id of the database and guid2 is the id of the ChartImageSessionBlock. To exploit the bug, we will change guid2 to the id of another session state that contains the binarydata that causes the error. So after completing the error triggering process, the next thing to do is figure out how to put any binary data into the session state table in the database.

[An article](https://www.zerodayinitiative.com/blog/2021/3/17/cve-2021-27076-a-replay-style-deserialization-attack-against-sharepoint) is published on the ZDI site about A previous vulnerability with code CVE-2021-27076 related to session state, using attachment mechanism on an infopath form. When starting to create a new item in an infopath list, the item will be registered with a session key of __itemId__. Next, when attaching a file to this new item, that file will be saved as a binary in the database with the key __attachmentId__. 

Any binary data will be in the attachment file, and __attachmentId__ is what we need to get to trigger the error. The problem is that when creating a new item in the infolist, only __itemId__ will be returned. Through building the lab, found that the value of __attachmentId__ is in the binarydata of the item, so we need to find a way to get __attachmentId__ through __itemId__. The ZDI article also showed how to solve this problem, which is to replay __itemId__ to FormServerAttachments.aspx, which will take the item's binarydata and return it as a file.

Here there are two directions to find the right request to FormServerAttachments.aspx, one is to try the functions, the other is to read the code and create the request yourself. The first option will be better because it will save time and I will also get a standard request to do, in case the function cannot be determined, it is mandatory to follow option 2 to read the code. Because the binary is returned as a file, the FileDownload function caught my eye.

```csharp
// Microsoft.Office.InfoPath.Server.Controls.FormServerAttachments.FileDownload(HttpContext) 
private static bool FileDownload(HttpContext context)
{
    string text = context.Request.QueryString["fid"];
    string text2 = context.Request.QueryString["sid"];
    string value = context.Request.QueryString["key"];
    string strA = context.Request.QueryString["dl"];
    int num = 0;
    string empty = string.Empty;
    if (string.IsNullOrEmpty(text) || string.IsNullOrEmpty(text2) || string.IsNullOrEmpty(value) || (string.Compare(strA, "fa", StringComparison.OrdinalIgnoreCase) != 0 && string.Compare(strA, "ip", StringComparison.OrdinalIgnoreCase) != 0))
    {
        ULS.SendTraceTag(1831874679U, ULSCat.msoulscat_formservices_runtime, ULSTraceLevel.Medium, "Invalid request incorrect or missing query strings: {0}", new object[]
        {
                    context.Request.Url.ToString()
        });
        return false;
    }
    using (new GlobalStorageContext(text))
    {
        try
        {
            SPSite spsite = SiteAndWebCache.Fetch().EnsureRequestSite();
            Solution solutionById = SolutionCache.GetSolutionById(spsite, new SolutionIdentity(text2));
            if (Canary.VerifyCanaryFromCookie(context, spsite, solutionById))
            {
                context.Response.Clear();
                context.Response.Cache.SetExpires(DateTime.Now.AddDays(2.0));
                using (BinaryWriter binaryWriter = new BinaryWriter(context.Response.OutputStream))
                {
                    Base64DataStorage.Base64DataItem item = null;
                    StreamUtils.DeserializeObjectsFromString(value, delegate (EnhancedBinaryReader binaryReader)
                    {
                        item = new Base64DataStorage.Base64DataItem(binaryReader);
                        DocumentChildState.StateInfo stateInfo = new DocumentChildState.StateInfo();
                        ((IBinaryDeserializable)stateInfo).Deserialize(binaryReader);
                        StateKey stateKey = StateKey.ParseKey(stateInfo.SerializedKey);
                        item.EnsureData(stateKey);
                    });
                    byte[] dataAsBytes = item.GetDataAsBytes();
                    using (Stream stream = new MemoryStream(dataAsBytes, false))
                    {
                        if (string.Compare(strA, "fa", StringComparison.OrdinalIgnoreCase) != 0)
                        {
                            context.Response.AppendHeader("Content-Disposition", "attachment;filename=\"image\"");
                            context.Response.AppendHeader("X-Download-Options", "noopen");
                            context.Response.ContentType = ImageUtils.GetContentType(dataAsBytes);
                            return InlinePicture.ReadInfoFromStream(binaryWriter, stream);
                        }
                        context.Response.ContentType = "application/octet-stream";
                        if (FileAttachment.ReadInfoFromStream(binaryWriter, out num, out empty, stream))
                        {
                            FilePathUtils.AddFileDownloadHttpHeader(context, empty);
                            return true;
                        }
                        return false;
                    }
                }
            }
            ULS.SendTraceTag(1831874680U, ULSCat.msoulscat_formservices_runtime, ULSTraceLevel.Verbose, "Can't verify canary from cookie for FileDownload");
            return false;
        }
        catch (InfoPathException)
        {
            ULS.SendTraceTag(1831874681U, ULSCat.msoulscat_formservices_runtime, ULSTraceLevel.Medium, "InfoPathException occurred downloading fileattachment or inline picture");
        }
    }
    return false;
}
```

Luckily the variables required for request are pretty obvious - __fid__, __sid__, __key__, __dl__. Let's dive a little deeper into the components, the following code is the error return condition

```csharp
// fid -> text, sid -> text2, key -> value, dl -> strA
if (string.IsNullOrEmpty(text) || string.IsNullOrEmpty(text2) || string.IsNullOrEmpty(value) || (string.Compare(strA, "fa", StringComparison.OrdinalIgnoreCase) != 0 && string.Compare(strA, "ip", StringComparison.OrdinalIgnoreCase) != 0))
```

The required params must be non-empty, where __dl__ must be either the string 'fa' or 'ip'.

```csharp
// fid -> text, sid -> text2, key -> value, dl -> strA
SPSite spsite = SiteAndWebCache.Fetch().EnsureRequestSite();
Solution solutionById = SolutionCache.GetSolutionById(spsite, new SolutionIdentity(text2));
if (Canary.VerifyCanaryFromCookie(context, spsite, solutionById))
{
    ...
}
```

This code does get the solutionId from __sid__ and authenticate with the infopath canary inside the cookie, for example a cookie like this:

```
_InfoPath_CanaryValueAGQX2G3RUCCXQRUNZHR3UB7IIEMSOL2MNFZXI4ZPORSXG5C7NFXGM327NRUXG5BPJF2GK3JPORSW24DMMF2GKLTYONXCWMKZLBZTE4TDI5WXC4ZSIIZGIUTINE4EI6DBGFWVKNKDLFZGSTJYLFNHE33VMJ5EGSLEOM=KBxeU4WXMZ3Yg8v0ZPZfAWcpoiLL/R3sfejthMFTfL1x9GqMoiIOMSS9XrT0gguJmdn0Yj2qw0gqlDJXT7X49A==|637806206864107501
```

This cookie has a key in the format '_InfoPath_CanaryValue'+ <suffix>. The main suffix is __sid__ to look for. Next is the code that gets the binary data from the session key.

```csharp
// fid -> text, sid -> text2, key -> value, dl -> strA
Base64DataStorage.Base64DataItem item = null;
StreamUtils.DeserializeObjectsFromString(value, delegate (EnhancedBinaryReader binaryReader)
{
    item = new Base64DataStorage.Base64DataItem(binaryReader);
    DocumentChildState.StateInfo stateInfo = new DocumentChildState.StateInfo();
    ((IBinaryDeserializable)stateInfo).Deserialize(binaryReader);
    StateKey stateKey = StateKey.ParseKey(stateInfo.SerializedKey);
    item.EnsureData(stateKey);
});
byte[] dataAsBytes = item.GetDataAsBytes();
```

The session state key will be retrieved from the __key__ variable, let's see the details of the DeserializeObjectsFromString function

```csharp
// fid -> text, sid -> text2, key -> value, dl -> strA
internal static void DeserializeObjectsFromString(string value, Action<EnhancedBinaryReader> readerMethod)
{
    using (Base64Stream base64Stream = new Base64Stream(value))
    {
        using (EnhancedBinaryReader enhancedBinaryReader = new EnhancedBinaryReader(base64Stream))
        {
            readerMethod(enhancedBinaryReader);
        }
    }
}
```

So __key__ needs to be in base64, see Base64DataStorage.Base64DataItem(binaryReader) function.

```csharp
// Microsoft.Office.InfoPath.Server.SolutionLifetime.Base64DataStorage.Base64DataItem.Base64DataItem(EnhancedBinaryReader)
internal Base64DataItem(EnhancedBinaryReader reader)
{
    this._state = (Base64ItemState)reader.ReadCompressedInt();
    this._sessionDataType = (Base64DataStorage.Base64DataItem.DataTypeInSessionState)reader.ReadCompressedInt();
    this._itemId = new Base64SerializationId(reader);
}
```

So the first 3 positions in the key's structure will be
- base64ItemState (int)
- dataTypeInSessionState (int)
- base64SerializationId (guid string)

let's see the function DocumentChildState.StateInfo.Deserialize(binaryReader)

```csharp
// DocumentChildState.StateInfo.Deserialize(binaryReader)
void IBinaryDeserializable.Deserialize(EnhancedBinaryReader reader)
{
    this._serializedKey = reader.ReadString();
    this._size = reader.ReadCompressedInt();
    this._version = reader.ReadCompressedInt();
}
```

so the next 3 positions in the key's structure will be
- serializedKey (string)
- size (int)
- version (int)

Next need to consider which components will be required to give the correct value, the part to get the session state key is as follows

```csharp
StateKey stateKey = StateKey.ParseKey(stateInfo.SerializedKey);
item.EnsureData(stateKey);
```
  
So the serializedKey will have the form guid1_guid2, where guid1 will be the database id and guid2 is the __itemId__ we put in, next see the item.EnsureData(stateKey) function

```csharp
// Microsoft.Office.InfoPath.Server.SolutionLifetime.Base64DataStorage.Base64DataItem.EnsureData(StateKey)
internal void EnsureData(StateKey stateKey)
{
    if (this.State == Base64ItemState.DelayLoad)
    {
        byte[] sessionData = StateManager.GetManager(HttpContext.Current).PeekState(stateKey); // get binary data from stateKey
        this.SetSessionData(sessionData);
        return;
    }
    if (this.State == Base64ItemState.Removed)
    {
        throw new InfoPathLocalizedException(InfoPathResourceManager.Ids.ServerGenericError, new string[0]);
    }
}
```
  
The first condition must be met to get binarydata from the database, so base64ItemState must have the value of enum Base64ItemState.DelayLoad, see inside Base64ItemState

```csharp
internal enum Base64ItemState
{
    NoChange,
    Updated,
    Removed,
    New,
    DelayLoad // 4
}
```
  
From there base64ItemState will take the value 4. Next look at the enum values of dataTypeInSessionState

```csharp
private enum DataTypeInSessionState
{
    Unknown,
    Utf8String,
    ByteArray  // 2
}
```

The data we need is stored in the session state table as binary data, so the value of dataTypeInSessionState should be 2. In summary, __key__ has the following structure.

![image](/assets/posts/cve-2022-22005/3.png)

After getting the binarydata, here is the code returned as a file  

```csharp
using (Stream stream = new MemoryStream(dataAsBytes, false))
{
    if (string.Compare(strA, "fa", StringComparison.OrdinalIgnoreCase) != 0)
    {
        context.Response.AppendHeader("Content-Disposition", "attachment;filename=\"image\"");
        context.Response.AppendHeader("X-Download-Options", "noopen");
        context.Response.ContentType = ImageUtils.GetContentType(dataAsBytes);
        return InlinePicture.ReadInfoFromStream(binaryWriter, stream);
    }
    context.Response.ContentType = "application/octet-stream";
    if (FileAttachment.ReadInfoFromStream(binaryWriter, out num, out empty, stream))
    {
        FilePathUtils.AddFileDownloadHttpHeader(context, empty);
        return true;
    }
    return false;
}
```

so __dl__ must have the value 'ip'. The variables sent to FormServerAttachments.aspx have the following form

![image](/assets/posts/cve-2022-22005/4.png)
  
## Exploit steps
After detailed analysis, the steps of exploiting the error are summarized as follows:
1. Create an infopath list on the site.
2. Open the form to create a new item on the list, save the __itemId__ from the response.
3. Attachment file contains the payload on that item, but don't press save so that the session state is kept in the database.
4. Put the itemId information obtained from step 2 into the request sent to FormServerAttachments.aspx, save the __attachmentId__ information from the response.
5. Include __attachmentId__ in the request to trigger deserialize in ChartPreviewImage.

## Permission
By default, a normal account has permission to create sub-sites and that account will have full permissions on the new site. Therefore, to exploit the bug, only the account needs basic permissions.
    
## Proof of Concept
[https://youtu.be/1Ckjh-uuNu4](https://youtu.be/1Ckjh-uuNu4)

# REFERENCES
[https://www.zerodayinitiative.com/blog/2021/3/17/cve-2021-27076-a-replay-style-deserialization-attack-against-sharepoint](https://www.zerodayinitiative.com/blog/2021/3/17/cve-2021-27076-a-replay-style-deserialization-attack-against-sharepoint)
