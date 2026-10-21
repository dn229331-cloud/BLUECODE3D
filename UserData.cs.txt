using System;
using System.Collections.Generic;

public enum UserRole
{
    Estudiante,
    Docente,
    Administrador
}

public enum UserPermission
{
    ConsultarPerfil,
    ModificarPerfil,
    RealizarSimulacion,
    RealizarEvaluacion,
    ConsultarResultadosPropios,

    ConsultarEstudiantes,
    CrearSimulacion,
    ConsultarResultadosEstudiantes,
    ExportarReportes,

    GestionarUsuarios,
    GestionarCasosClinicos,
    GestionarMatrizEvaluacion,
    ConsultarEventos,
    ModificarSistema,
    CrearRespaldos
}

[Serializable]
public class UserData
{
    public string nombreCompleto;
    public string codigoInstitucional;
    public string semestre;

    // Nunca guardar la contraseÃ±a en texto plano.
    public string passwordHash;
    public string passwordSalt;

    public bool aceptoTerminos;
    public UserRole rol;

    public UserData(
        string nombre,
        string codigo,
        string semestreUsuario,
        string hash,
        string salt,
        bool terminos,
        UserRole rolUsuario)
    {
        nombreCompleto = nombre;
        codigoInstitucional = codigo;
        semestre = semestreUsuario;
        passwordHash = hash;
        passwordSalt = salt;
        aceptoTerminos = terminos;
        rol = rolUsuario;
    }

    public List<UserPermission> ObtenerPermisos()
    {
        switch (rol)
        {
            case UserRole.Estudiante:
                return new List<UserPermission>
                {
                    UserPermission.ConsultarPerfil,
                    UserPermission.ModificarPerfil,
                    UserPermission.RealizarSimulacion,
                    UserPermission.RealizarEvaluacion,
                    UserPermission.ConsultarResultadosPropios
                };

            case UserRole.Docente:
                return new List<UserPermission>
                {
                    UserPermission.ConsultarPerfil,
                    UserPermission.ConsultarEstudiantes,
                    UserPermission.CrearSimulacion,
                    UserPermission.ConsultarResultadosEstudiantes,
                    UserPermission.ExportarReportes
                };

            case UserRole.Administrador:
                return new List<UserPermission>
                {
                    UserPermission.ConsultarPerfil,
                    UserPermission.GestionarUsuarios,
                    UserPermission.GestionarCasosClinicos,
                    UserPermission.GestionarMatrizEvaluacion,
                    UserPermission.ConsultarEventos,
                    UserPermission.ModificarSistema,
                    UserPermission.CrearRespaldos
                };

            default:
                return new List<UserPermission>();
        }
    }
}