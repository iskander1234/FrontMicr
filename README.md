// Services/Interfaces/IFileService.cs
using System.IO;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;

namespace BpmBaseApi.Services.Interfaces
{
    /// <summary>
    /// Клиент для работы с внешним файловым сервисом:
    ///   - загрузка
    ///   - скачивание
    ///   - удаление
    /// </summary>
    public interface IFileService
    {
        /// <summary>
        /// Загружает файл на файловый сервер, возвращает имя сохранённого файла.
        /// </summary>
        Task<string> UploadAsync(IFormFile file, CancellationToken ct);

        /// <summary>
        /// Скачивает файл по имени, возвращает поток содержимого.
        /// </summary>
        Task<Stream> GetAsync(string fileName, CancellationToken ct);

        /// <summary>
        /// Удаляет файл по имени.
        /// </summary>
        Task DeleteAsync(string fileName, CancellationToken ct);
    }
}


// Services/Implementations/FileService.cs
using System;
using System.IO;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Threading;
using System.Threading.Tasks;
using BpmBaseApi.Services.Interfaces;
using Microsoft.AspNetCore.Http;

namespace BpmBaseApi.Services.Implementations
{
    public class FileService : IFileService
    {
        private readonly HttpClient _http;

        public FileService(HttpClient http)
        {
            _http = http;
        }

        public async Task<string> UploadAsync(IFormFile file, CancellationToken ct)
        {
            using var content = new MultipartFormDataContent();
            using var stream = file.OpenReadStream();
            var fileContent = new StreamContent(stream);
            fileContent.Headers.ContentType = MediaTypeHeaderValue.Parse(file.ContentType);

            // "file" — имя поля, как ожидает внешний сервис
            content.Add(fileContent, "file", file.FileName);

            var resp = await _http.PostAsync("upload", content, ct);
            resp.EnsureSuccessStatusCode();

            // Предположим, что сервис возвращает в теле JSON { "fileName": "..." }
            var dto = await resp.Content.ReadAsAsync<UploadResultDto>(ct);
            return dto.FileName;
        }

        public async Task<Stream> GetAsync(string fileName, CancellationToken ct)
        {
            var resp = await _http.GetAsync(fileName, ct);
            resp.EnsureSuccessStatusCode();
            return await resp.Content.ReadAsStreamAsync(ct);
        }

        public async Task DeleteAsync(string fileName, CancellationToken ct)
        {
            // путь DELETE delete/{fileName}
            var resp = await _http.DeleteAsync($"delete/{Uri.EscapeDataString(fileName)}", ct);
            resp.EnsureSuccessStatusCode();
        }

        private class UploadResultDto
        {
            public string FileName { get; set; }
        }
    }
}


// В Program.cs, после builder.Services…
builder.Services
    .AddHttpClient<IFileService, FileService>(client => {
        client.BaseAddress = new Uri("http://172.31.120.61:8020/files/");
    });


// WebAPI/Controllers/FilesController.cs
using System.Threading;
using System.Threading.Tasks;
using BpmBaseApi.Services.Interfaces;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;

namespace BpmBaseApi.WebAPI.Controllers
{
    [ApiController]
    [Route("api/files")]
    public class FilesController : ControllerBase
    {
        private readonly IFileService _fileService;

        public FilesController(IFileService fileService)
        {
            _fileService = fileService;
        }

        /// <summary>Загрузить файл</summary>
        [HttpPost("upload")]
        public async Task<IActionResult> Upload(
            IFormFile file,
            CancellationToken ct)
        {
            var fileName = await _fileService.UploadAsync(file, ct);
            return Ok(new { fileName });
        }

        /// <summary>Скачать файл</summary>
        [HttpGet("{fileName}")]
        public async Task<IActionResult> Download(
            [FromRoute] string fileName,
            CancellationToken ct)
        {
            var stream = await _fileService.GetAsync(fileName, ct);
            return File(stream, "application/octet-stream", fileName);
        }

        /// <summary>Удалить файл</summary>
        [HttpDelete("delete/{fileName}")]
        public async Task<IActionResult> Delete(
            [FromRoute] string fileName,
            CancellationToken ct)
        {
            await _fileService.DeleteAsync(fileName, ct);
            return NoContent();
        }
    }
}



// Services/Implementations/FileService.cs
using System;
using System.IO;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Net.Http.Json;
using System.Threading;
using System.Threading.Tasks;
using BpmBaseApi.Services.Interfaces;
using Microsoft.AspNetCore.Http;

namespace BpmBaseApi.Services.Implementations
{
    /// <summary>
    /// Реализация IFileService на основе HttpClient.
    /// </summary>
    public class FileService : IFileService
    {
        private readonly HttpClient _http;

        public FileService(HttpClient http)
        {
            _http = http;
        }

        public async Task<string> UploadAsync(IFormFile file, CancellationToken ct)
        {
            using var content = new MultipartFormDataContent();
            using var stream = file.OpenReadStream();
            var fileContent = new StreamContent(stream);
            fileContent.Headers.ContentType = MediaTypeHeaderValue.Parse(file.ContentType);

            // "file" — имя поля, как ожидает внешний сервис
            content.Add(fileContent, "file", file.FileName);

            var resp = await _http.PostAsync("upload", content, ct);
            resp.EnsureSuccessStatusCode();

            // Читаем ответ JSON { "fileName": "..." }
            var dto = await resp.Content.ReadFromJsonAsync<UploadResultDto>(cancellationToken: ct);
            if (dto == null || string.IsNullOrWhiteSpace(dto.FileName))
                throw new InvalidOperationException("Не удалось получить имя файла от сервиса");

            return dto.FileName;
        }

        public async Task<Stream> GetAsync(string fileName, CancellationToken ct)
        {
            // GET http://.../files/{fileName}
            var resp = await _http.GetAsync(fileName, ct);
            resp.EnsureSuccessStatusCode();
            return await resp.Content.ReadAsStreamAsync(ct);
        }

        public async Task DeleteAsync(string fileName, CancellationToken ct)
        {
            // DELETE http://.../files/delete/{fileName}
            var url = $"delete/{Uri.EscapeDataString(fileName)}";
            var resp = await _http.DeleteAsync(url, ct);
            resp.EnsureSuccessStatusCode();
        }

        private class UploadResultDto
        {
            public string FileName { get; set; } = default!;
        }
    }
}
