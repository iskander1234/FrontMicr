1.Upload
curl --location 'http:// 172.31.120.61:8020/files/upload' \\

{
  "file": "string"
}




--form 'file=@"/C:/Users/Kushtar/Pictures/photo\_2023-09-07\_11-57-52.jpg"'

2. Get file
curl --location 'http:// 172.31.120.61:8020/files/1753855905928\_photo\_2023-09-07\_11-57-52.jpg'

fileName *
string
(path)

3\. Delete file

curl --location --request DELETE 'http:// 172.31.120.61:8020/files/delete/1753855721993\_photo\_2023-09-07\_11-57-52.jpg'


fileName *
string
(path)
