using TMPro;
using UnityEngine;
using UnityEngine.UI;

public class AuthUI : MonoBehaviour
{
    [Header("Paneles")]
    [SerializeField] private GameObject panelLogin;
    [SerializeField] private GameObject panelRegistro;

    [Header("Campos del registro")]
    [SerializeField] private TMP_InputField nombreInput;
    [SerializeField] private TMP_InputField codigoRegistroInput;
    [SerializeField] private TMP_InputField semestreInput;
    [SerializeField] private TMP_InputField contrasenaRegistroInput;
    [SerializeField] private Toggle aceptarTerminosToggle;

    [Header("Campos del login")]
    [SerializeField] private TMP_InputField usuarioLoginInput;
    [SerializeField] private TMP_InputField contrasenaLoginInput;

    private void Start()
    {
        MostrarLogin();
    }

    public void Registrar()
    {
        if (AuthManager.Instance == null)
        {
            Debug.LogError("AuthManager no esta disponible.");
            return;
        }

        bool registrado = AuthManager.Instance.RegistrarEstudiante(
            nombreInput.text,
            codigoRegistroInput.text,
            semestreInput.text,
            contrasenaRegistroInput.text,
            aceptarTerminosToggle.isOn
        );

        if (registrado)
        {
            Debug.Log("Ya puedes iniciar sesion.");

            LimpiarRegistro();
            MostrarLogin();
        }
    }

    public void IniciarSesion()
    {
        if (AuthManager.Instance == null)
        {
            Debug.LogError("AuthManager no esta disponible.");
            return;
        }

        bool autenticado = AuthManager.Instance.IniciarSesion(
            usuarioLoginInput.text,
            contrasenaLoginInput.text
        );

        if (autenticado)
        {
            // Por ahora no tenemos otras vistas.
            // El mensaje de bienvenida se imprime
            // desde AuthManager.

            contrasenaLoginInput.text = "";
            panelLogin.SetActive(false);
            panelRegistro.SetActive(false);
        }
    }

    public void MostrarRegistro()
    {
        panelLogin.SetActive(false);
        panelRegistro.SetActive(true);
    }

    public void MostrarLogin()
    {
        panelLogin.SetActive(true);
        panelRegistro.SetActive(false);
    }

    public void CerrarSesion()
    {
        if (AuthManager.Instance != null)
            AuthManager.Instance.CerrarSesion();

        usuarioLoginInput.text = "";
        contrasenaLoginInput.text = "";

        MostrarLogin();
    }

    private void LimpiarRegistro()
    {
        nombreInput.text = "";
        codigoRegistroInput.text = "";
        semestreInput.text = "";
        contrasenaRegistroInput.text = "";
        aceptarTerminosToggle.isOn = false;
    }
}