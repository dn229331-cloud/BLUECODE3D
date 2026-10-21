using System;
using System.Collections.Generic;
using System.IO;
using System.Security.Cryptography;
using System.Text;
using UnityEngine;

public class AuthManager : MonoBehaviour
{
    public static AuthManager Instance { get; private set; }

    private const int Iteraciones = 100000;
    private const int LongitudHash = 32;

    private List<UserData> usuarios = new List<UserData>();

    public UserData UsuarioActual { get; private set; }

    private string RutaArchivo =>
        Path.Combine(Application.persistentDataPath, "usuarios.json");

    [Serializable]
    private class UserDatabase
    {
        public List<UserData> usuarios = new List<UserData>();
    }

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }

        Instance = this;
        DontDestroyOnLoad(gameObject);

        CargarUsuarios();
    }

    public bool RegistrarEstudiante(
        string nombre,
        string codigo,
        string semestre,
        string contrasena,
        bool aceptoTerminos)
    {
        nombre = nombre == null ? "" : nombre.Trim();
        codigo = codigo == null ? "" : codigo.Trim();
        semestre = semestre == null ? "" : semestre.Trim();

        if (string.IsNullOrWhiteSpace(nombre) ||
            string.IsNullOrWhiteSpace(codigo) ||
            string.IsNullOrWhiteSpace(semestre) ||
            string.IsNullOrEmpty(contrasena))
        {
            Debug.LogError("Completa todos los campos.");
            return false;
        }

        if (nombre.Length > 100 ||
            codigo.Length > 50 ||
            semestre.Length > 20)
        {
            Debug.LogError(
                "Uno o varios campos superan el limite permitido.");
            return false;
        }

        int numeroSemestre;

        if (!int.TryParse(semestre, out numeroSemestre) ||
            numeroSemestre < 1 ||
            numeroSemestre > 20)
        {
            Debug.LogError(
                "El semestre debe ser un numero entre 1 y 20.");
            return false;
        }

        if (contrasena.Length < 8)
        {
            Debug.LogError(
                "La contrasena debe tener al menos 8 caracteres.");
            return false;
        }

        if (!aceptoTerminos)
        {
            Debug.LogError(
                "Debes aceptar los terminos y condiciones.");
            return false;
        }

        foreach (UserData usuario in usuarios)
        {
            if (string.Equals(
                usuario.codigoInstitucional,
                codigo,
                StringComparison.OrdinalIgnoreCase))
            {
                Debug.LogError(
                    "Ya existe un usuario con ese codigo institucional.");
                return false;
            }
        }

        byte[] salt = new byte[16];

        using (RandomNumberGenerator rng =
               RandomNumberGenerator.Create())
        {
            rng.GetBytes(salt);
        }

        string saltBase64 = Convert.ToBase64String(salt);
        string hash = GenerarHash(contrasena, salt);

        UserData nuevoUsuario = new UserData(
            nombre,
            codigo,
            semestre,
            hash,
            saltBase64,
            aceptoTerminos,
            UserRole.Estudiante
        );

        usuarios.Add(nuevoUsuario);

        if (!GuardarUsuarios())
        {
            usuarios.Remove(nuevoUsuario);
            Debug.LogError("No fue posible guardar el registro.");
            return false;
        }

        Debug.Log("Registro exitoso para: " + nombre);

        return true;
    }

    public bool IniciarSesion(string codigo, string contrasena)
    {
        codigo = codigo == null ? "" : codigo.Trim();

        if (string.IsNullOrWhiteSpace(codigo) ||
            string.IsNullOrEmpty(contrasena))
        {
            Debug.LogError(
                "Ingresa el usuario y la contrasena.");
            return false;
        }

        foreach (UserData usuario in usuarios)
        {
            if (string.Equals(
                usuario.codigoInstitucional,
                codigo,
                StringComparison.OrdinalIgnoreCase))
            {
                byte[] salt;

                try
                {
                    salt = Convert.FromBase64String(
                        usuario.passwordSalt);
                }
                catch (FormatException)
                {
                    Debug.LogError(
                        "Datos de autenticacion invalidos.");
                    return false;
                }

                string hashIngresado =
                    GenerarHash(contrasena, salt);

                if (!HashesIguales(
                    hashIngresado,
                    usuario.passwordHash))
                {
                    Debug.LogError("Contrasena incorrecta.");
                    return false;
                }

                UsuarioActual = usuario;

                Debug.Log(
                    "Bienvenido " + usuario.nombreCompleto +
                    " (" + usuario.rol + ")");

                Debug.Log("Permisos asignados:");

                foreach (UserPermission permiso
                         in usuario.ObtenerPermisos())
                {
                    Debug.Log("- " + permiso);
                }

                return true;
            }
        }

        Debug.LogError("Usuario no encontrado.");

        return false;
    }

    public bool TienePermiso(UserPermission permiso)
    {
        if (UsuarioActual == null)
            return false;

        return UsuarioActual.ObtenerPermisos().Contains(permiso);
    }

    public void CerrarSesion()
    {
        if (UsuarioActual == null)
            return;

        Debug.Log(
            "Sesion cerrada: " +
            UsuarioActual.codigoInstitucional);

        UsuarioActual = null;
    }

    private string GenerarHash(string contrasena, byte[] salt)
    {
        using (var derivador = new Rfc2898DeriveBytes(
            contrasena,
            salt,
            Iteraciones,
            HashAlgorithmName.SHA256))
        {
            return Convert.ToBase64String(
                derivador.GetBytes(LongitudHash));
        }
    }

    private bool HashesIguales(string a, string b)
    {
        try
        {
            byte[] bytesA = Convert.FromBase64String(a);
            byte[] bytesB = Convert.FromBase64String(b);

            return CryptographicOperations.FixedTimeEquals(
                bytesA, bytesB);
        }
        catch (FormatException)
        {
            return false;
        }
    }

    private void CargarUsuarios()
    {
        if (!File.Exists(RutaArchivo))
        {
            usuarios = new List<UserData>();

            Debug.Log(
                "No existe un archivo de usuarios. " +
                "Se inicia una lista vacia.");

            Debug.Log("Ruta del archivo: " + RutaArchivo);

            return;
        }

        try
        {
            string json = File.ReadAllText(
                RutaArchivo, Encoding.UTF8);

            UserDatabase baseDatos =
                JsonUtility.FromJson<UserDatabase>(json);

            usuarios = baseDatos != null &&
                       baseDatos.usuarios != null
                ? baseDatos.usuarios
                : new List<UserData>();

            Debug.Log(
                "Usuarios locales cargados: " + usuarios.Count);

            Debug.Log("Ruta del archivo: " + RutaArchivo);
        }
        catch (Exception e)
        {
            usuarios = new List<UserData>();

            Debug.LogError(
                "No se pudo leer el archivo local: " + e.Message);
        }
    }

    private bool GuardarUsuarios()
    {
        try
        {
            UserDatabase baseDatos = new UserDatabase();
            baseDatos.usuarios = usuarios;

            string json = JsonUtility.ToJson(baseDatos, true);

            string temporal = RutaArchivo + ".tmp";

            File.WriteAllText(temporal, json, Encoding.UTF8);

            if (File.Exists(RutaArchivo))
                File.Delete(RutaArchivo);

            File.Move(temporal, RutaArchivo);

            Debug.Log(
                "Usuarios guardados correctamente: " +
                usuarios.Count);

            Debug.Log("Ruta del archivo: " + RutaArchivo);

            return true;
        }
        catch (Exception e)
        {
            Debug.LogError(
                "Error al guardar usuarios: " + e.Message);

            return false;
        }
    }

    // Diagnostico: muestra los usuarios cargados en memoria.
    // Se ejecuta desde el menu contextual del componente.
    [ContextMenu("Depuracion/Mostrar usuarios registrados")]
    private void MostrarUsuariosRegistrados()
    {
        Debug.Log("========== USUARIOS REGISTRADOS ==========");
        Debug.Log("Archivo: " + RutaArchivo);
        Debug.Log("Total en memoria: " + usuarios.Count);

        if (usuarios.Count == 0)
        {
            Debug.LogWarning(
                "No hay usuarios cargados en memoria.");

            return;
        }

        for (int i = 0; i < usuarios.Count; i++)
        {
            UserData usuario = usuarios[i];

            if (usuario == null)
            {
                Debug.LogWarning(
                    "Usuario nulo en la posicion " + i);

                continue;
            }

            Debug.Log(
                $"Usuario #{i + 1}\n" +
                $"Nombre: {usuario.nombreCompleto}\n" +
                $"Codigo: '{usuario.codigoInstitucional}'\n" +
                $"Semestre: {usuario.semestre}\n" +
                $"Rol: {usuario.rol}\n" +
                $"Acepto terminos: {usuario.aceptoTerminos}"
            );
        }

        Debug.Log("========== FIN DEL LISTADO ==========");
    }
}