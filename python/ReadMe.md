为了让用户获得文档解析引擎返回的丰富版面元素，我们开发了一系列的sdk函数，包括目录树、公式、表格、图片、全文markdown等结果的获取函数。一些场景中，部分用户希望通过api进行调用，还有一部分用户希望能够直接可视化文件解析的结果，且可以进行编辑修正，此次我们也将textin.com上面的前端可视化组件进行了开源，请用户批评指正。

1、SDK功能介绍
ParseX是一套标准的多平台支持的python sdk，帮助开发者解析pdf_to_markdownRestful API返回结果，获取对应的版面元素的数据结构。开发者只需在终端安装对应的依赖就可以使用。

```
pip install TextInParseX
```

首先，你要在textin开通文档解析服务，然后可以在 https://www.textin.com/console/dashboard/setting 获得api_id和secret_code
你可以通过ParseX直接调用url获取解析对象

```
import TextInParseX as px

# 初始化解析器
app_id = "#############################"     #填入你的textin的api_id和secret——code
secret_code = "#############################"

parseX_client = px.ParseXClient(app_id, secret_code)

pdf_file_path = "example.pdf" #你的本地文件路径

#通过ParseGenious直接调用url获取解析对象
result = parseX_client.begin_analyze_document_from_url(pdf_file_path)

```

也可以参考textin.com的restful api调用，通过python，curl，或者postman工具获得api的原始json文件，再通过ParseX解析json文件获得解析对象。

```
import TextInParseX as px
import json

json_file = 'test_json/example.json'
with open(json_file, 'r') as fr:
    json_result = json.load(fr)
    
parseX_client = px.ParseXClient()
result = parseX_client.begin_analyze_document_from_json(json_result)

#或者直接输入json文件
result = parseX_client.begin_analyze_document_from_file(json_file)
```

获得文档的markdown 信息，所有文本、表格、段落，图片信息
```
print('Markdown:')
print(result.all_markdown)
print("\n")

print("All text in document:")
parseX_client.print_all_elements(result.all_text, 0, 1000)
print("\n")
print(f"Total tables in document: {len(result.all_tables)}")
for index, table in enumerate(result.all_tables):
    print(f"Table {index }:")
    parseX_client.print_all_elements(table)
    print("\n")
    
print(f"Total paragraphs in document: {len(result.all_paragraphs)}")
for p_idx, each_paragraph in enumerate(result.all_paragraphs):
    print(f"\n--- Paragraph {p_idx }/{len(result.all_paragraphs)} ---")
    print(f"Paragraph position: {each_paragraph.pos}")
    for l_idx, each_line in enumerate(each_paragraph.lines):
        print(f"  Line {l_idx }/{len(each_paragraph.lines)}")
        print(f"    Line positions: {each_line.pos}")
        print(f"  Line text: {each_line.text}")
    
print(f"Total images in document: {len(result.all_images)}")
for index, image in enumerate(result.all_images):
    print(f"Image {index}:")
    parseX_client.print_all_elements(image)
    print("\n")

all_images_cv_mat = result.get_all_images_cv_mat()
print(f"Total images (as cv::Mat) in document: {len(all_images_cv_mat)}")
for idx, mat in enumerate(all_images_cv_mat):
    print(f"Image {idx } (cv::Mat) shape: {mat.shape}")
```

分别获取每页的表格信息、图片信息，段落纯文本信息，段落坐标，每行的信息

```
for page in result.pages:
    print(f"=== Page {page.page_id } ===")
    print("\n")
    
    for index, table in enumerate(page.tables):
        print(f"Table {index }:")
        parseX_client.print_all_elements(table)
        print("\n")

    for index, image in enumerate(page.images):
        print(f"Image {index }:")
        parseX_client.print_all_elements(image)  # 限定只能打印前50(默认)个字符
        print("\n")
    
    images_cv_mat = page.get_images_cv_mat()
    print(f"Total images (as cv::Mat) in page {page.page_id }: {len(images_cv_mat)}")
    for idx, mat in enumerate(images_cv_mat):
        print(f"Image {idx } (cv::Mat) shape: {mat.shape}")
    print("\n")
   
    print("Text:")
    parseX_client.print_all_elements(page.paragraph_text, 0, 1000)  # 限定只能打印前1000个字符
    print("\n")
    # 获取当前页的段落
    
    print(f"Total paragraphs: {len(page.paragraphs)}")
    for p_idx, each_paragraph in enumerate(page.paragraphs):
        print(f"\n--- Paragraph {p_idx }/{len(page.paragraphs)} ---")
        print(f"Paragraph position: {each_paragraph.pos}")
        for l_idx, each_line in enumerate(each_paragraph.lines):
            print(f"  Line {l_idx }/{len(each_paragraph.lines)}")
            print(f"    Line positions: {each_line.pos}")
            print(f"  Line text: {each_line.text}")
    print('Finished getting paragraphs')
    print("\n\n")
```

画出每页的表格、图片、段落、行的框

```
    for page in result.pages:
        page_img = parseX_client.download_image_from_url(download_image_url, page.image_id)
        for index, table in enumerate(page.tables):
            cells = table.cells
            for each_cell in cells:
                cv2.rectangle(page_img, (each_cell.pos[0], each_cell.pos[1]), (each_cell.pos[4], each_cell.pos[5]), (0, 0, 255), 1)
            
            for index, image in enumerate(page.images):
                cv2.rectangle(page_img, (image.pos[0], image.pos[1]), (image.pos[4], image.pos[5]), (0, 255, 255), 1)

            for p_idx, each_paragraph in enumerate(page.paragraphs):
                cv2.rectangle(page_img, (each_paragraph.pos[0], each_paragraph.pos[1]), (each_paragraph.pos[4], each_paragraph.pos[5]), (0, 255, 0), 5)
                for l_idx, each_line in enumerate(each_paragraph.lines):
                    cv2.rectangle(page_img, (each_line.pos[0], each_line.pos[1]), (each_line.pos[4], each_line.pos[5]), (255, 0, 0), 1)

        cv2.imwrite('image_result_{}.jpg'.format(page.page_id), page_img)
```

获取表格图片

```
    for page in result.pages:
        page_img = parseX_client.download_image_from_url(download_image_url, page.image_id)
        if page_img is None:
            continue

        for index, table in enumerate(page.tables):
            x1, y1 = table.pos[0], table.pos[1]
            x2, y2 = table.pos[4], table.pos[5]
            table_img = page_img[y1:y2, x1:x2]
            cv2.imwrite('page_{}_table_{}.jpg'.format(page.page_id, index), table_img)
```

将table转成excel表格
```
parseX_client.save_tables_as_excel(result.all_tables, 'tables_result.xlsx')
``` 