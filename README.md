
using UnityEngine;
using UnityEngine.UI;  // Image component'i için
using System.Collections;
using System.Collections.Generic;
using TMPro;

public class PlayerMovement : MonoBehaviour
{
    public CharacterController controller;
    public Image darkenImage; // Ekranı karartacak olan Image
    public TMP_Text winUIText; // Kazanma yazısı

    private bool isTakingDamage;

    // Sağlık sistemi  
    public float health = 100f;  // Oyuncunun sağlığı  
    public float damageAmount = 10f;  // Zarar miktarı  

    public float walkSpeed;
    public float runSpeed;
    public float speed = 12f;

    // Nesne alma  
    public float pickupRange = 2f;
    public LayerMask pickupMask;
    [SerializeField] private Transform itemHolder;
    private GameObject currentItem;
    public List<string> inventory = new List<string>();

    // Yer çekimi  
    public float gravity = -9.81f;
    Vector3 velocity;
    bool isGrounded;

    // Yerde durma kontrolü  
    public Transform GroundCheck;
    public float GroundDistance = 0.4f;
    public LayerMask GroundMask;
    public float JumpHeight;

    // Hızlanma ve yavaşlama  
    public float acceleration = 5f;
    public float deceleration = 5f;
    public float currentSpeed;

    // Ölüm anında ekran kararması  
    private bool isDead = false;  // Oyuncu öldü mü?
    private int currentItemIndex = -1;
    private bool isHoldingItem = false; // Nesne tutuluyor mu?

    //Ekrana yazı yazdırma
    public TMP_Text inventoryUIText; // TMPro;

    private List<string> collectedRocks = new List<string>(); // Toplanan taşlar
    private string[] requiredRocks = { "GreenRock", "YellowRock", "BlueRock", "RedRock" }; // Gereken taşlar

    private bool gameWon = false; // Oyunu kazandı mı?

    void Start()
    {
        currentSpeed = walkSpeed;

        inventoryUIText.text = ""; // Metni başta boş bırakıyoruz
        winUIText.text = ""; // Oyunun başında kazandınız yazmasın
    }

    void Update()
    {
        // Sağlık sıfır olduğunda ekran kararmaya başlasın
        if (health <= 0 && !isDead)
        {
            isDead = true;
            StartCoroutine(DarkenScreen());  // Sağlık sıfırlandığında ekranı karartmaya başla
        }

        if (isDead)
        {
            // Eğer oyuncu öldüyse ekran kararmaya başlar, hareket engellenir
            return;
        }

        // Yere basma kontrolü
        isGrounded = Physics.CheckSphere(GroundCheck.position, GroundDistance, GroundMask);
        if (isGrounded && velocity.y < 0)
        {
            velocity.y = -2f;
        }

        float x = Input.GetAxis("Horizontal");
        float z = Input.GetAxis("Vertical");

        if (Input.GetKey(KeyCode.LeftShift) && (x != 0 || z != 0))
        {
            currentSpeed = Mathf.MoveTowards(currentSpeed, runSpeed, acceleration * Time.deltaTime);
        }
        else
        {
            currentSpeed = Mathf.MoveTowards(currentSpeed, walkSpeed, deceleration * Time.deltaTime);
        }

        Vector3 move = transform.right * x + transform.forward * z;
        controller.Move(move * currentSpeed * Time.deltaTime);

        if (Input.GetButtonDown("Jump") && isGrounded)
        {
            velocity.y = Mathf.Sqrt(JumpHeight * -2f * gravity);
        }

        velocity.y += gravity * Time.deltaTime;
        controller.Move(velocity * Time.deltaTime);

        if (Input.GetKeyDown(KeyCode.E)) // E tuşuna basıldığında nesne al
        {
            PickupItem();
        }

        if (Input.GetKeyDown(KeyCode.Q)) // Q tuşuna basıldığında nesneyi bırak
        {
            DropItem();
        }
    }

    // Zarar veren alanda oyuncu girince zarar verme
    private void OnTriggerEnter(Collider other)
    {
        // Eğer tetiklenen nesne "DamageArea" tag'ına sahipse
        if (other.CompareTag("DamageArea"))
        {
            // Zarar ver
            TakeDamage(damageAmount);
            Debug.Log("Zarar alındı!");
        }
    }

    // Zarar alma fonksiyonu
    public void TakeDamage(float damage)
    {
        health -= damage; // Sağlık değerini azalt
        Debug.Log("Sağlık: " + health);
        if (health <= 0)
        {
            health = 0; // Sağlık sıfır olduğunda sıfırla  
            Die(); // Sağlık sıfır olduğunda ölüme sebep ol
        }
    }

    // Ölüm fonksiyonu
    void Die()
    {
        // Ölüm olayları (örneğin, karakterin oyun dışı kalması, oyun yeniden başlatma, can kaybı)
        Debug.Log("Öldün!");
    }

    // Envanteri güncelle
    void UpdateInventoryUI()
    {
        // Envanteri güncelle
    }

    // Nesne alma
    void PickupItem()
    {
        // Eğer mevcut bir nesne varsa, bırak  
        if (currentItem != null)
        {
            DropItem(); // Mevcut nesneyi bırakın  
        }

        // Yeni nesne alma işlemi  
        Collider[] hitColliders = Physics.OverlapSphere(transform.position, pickupRange, pickupMask);
        foreach (var hitCollider in hitColliders)
        {
            // Mevcut nesne yoksa yeni nesneyi al  
            currentItem = hitCollider.gameObject;
            inventory.Add(currentItem.name);
            collectedRocks.Add(currentItem.name); // Taşı topladıkça listeye ekle
            currentItem.transform.SetParent(itemHolder);
            currentItem.transform.localPosition = Vector3.zero;
            currentItem.transform.localRotation = Quaternion.identity;

            // Rigidbody varsa yer çekimi ile etkileyin  
            Rigidbody rb = currentItem.GetComponent<Rigidbody>();
            if (rb != null)
            {
                rb.isKinematic = true; // Mevcut nesneyi tutarken kinematik olmalı  
            }

            UpdateInventoryUI(currentItem.name);
            CheckForWin(); // Oyunu kazanıp kazanmadığını kontrol et
        }
    }

    void UpdateInventoryUI(string newItem)
    {
        inventoryUIText.text = $"Aldığınız Taş: {newItem}\nCanavar sizi yakalamadan çabuk diğer taşları bulun";
        StartCoroutine(HideInventoryTextAfterDelay(5f)); // 5 saniye sonra gizle
    }

    IEnumerator HideInventoryTextAfterDelay(float delay)
    {
        yield return new WaitForSeconds(delay);
        inventoryUIText.text = "";
    }

    void GameWin()
    {
        gameWon = true;

        // Oyuncu tüm taşları topladığında yazıyı kırmızı renkte göster
        winUIText.color = Color.red;
        winUIText.text = "Tüm taşları topladınız! Kazandınız!";  // Oyunun bitiş yazısını göster

        // Yazıyı ekranın tam ortasına yerleştirmek için
        winUIText.alignment = TextAlignmentOptions.Center;  // Yazıyı ortalar
        winUIText.rectTransform.anchorMin = new Vector2(0.5f, 0.5f);  // Ekranın ortası
        winUIText.rectTransform.anchorMax = new Vector2(0.5f, 0.5f);  // Ekranın ortası
        winUIText.rectTransform.anchoredPosition = Vector2.zero;  // Ortaya yerleştir

        // Oyuncunun hareketini engelle
        controller.enabled = false;  // Player controller'ı devre dışı bırak

        // 3 saniye sonra ekranı karart ve kazandınız yazısını göster
        StartCoroutine(DarkenAndDisplayWinText());
    }

    IEnumerator DarkenAndDisplayWinText()
    {
        // Ekranı 4 saniyede karart
        float targetAlpha = 1f;  // Tam kararma
        float duration = 4f; // 4 saniye içinde karartma
        float currentAlpha = darkenImage.color.a; // Mevcut opaklık

        float timeElapsed = 0f;

        // Ekran kararmaya başlasın
        while (timeElapsed < duration)
        {
            timeElapsed += Time.deltaTime;
            float alpha = Mathf.Lerp(currentAlpha, targetAlpha, timeElapsed / duration);  // Yavaşça karart
            darkenImage.color = new Color(0, 0, 0, alpha);  // Siyah rengini ve alfa değerini güncelle
            yield return null;
        }

        // Ekran karardıktan sonra oyun bitti
        Debug.Log("Oyun bitti! Kazandınız!");
    }

    void CheckForWin()
    {
        foreach (string rock in requiredRocks)
        {
            if (!collectedRocks.Contains(rock))
            {
                Debug.Log("Eksik taş var: " + rock);
                return; // Gereken bir taş eksikse kazanma fonksiyonu çağrılmaz
            }
        }
        Debug.Log("Tüm taşlar toplandı. Oyunu kazanıyorsunuz!");
        GameWin(); // Eğer tüm taşlar toplanmışsa oyunu kazan
    }

    // Nesneyi bırakma
    void DropItem()
    {
        if (currentItem != null)
        {
            currentItem.transform.SetParent(null);
            Rigidbody rb = currentItem.GetComponent<Rigidbody>();
            if (rb == null) rb = currentItem.AddComponent<Rigidbody>();
            rb.isKinematic = false;
            rb.AddForce(transform.forward * 2f, ForceMode.Impulse);
            currentItem = null;
        }
    }

    // Ekranı karartacak Coroutine
    private IEnumerator DarkenScreen()
    {
        float targetAlpha = 1f;  // Hedef opaklık (tam kararmış)
        float duration = 2f; // 2 saniye içinde ekranın kararmasını sağla
        float currentAlpha = darkenImage.color.a; // Mevcut opaklık

        float timeElapsed = 0f;

        // Ekran kararmaya başlasın
        while (timeElapsed < duration)
        {
            timeElapsed += Time.deltaTime;
            float alpha = Mathf.Lerp(currentAlpha, targetAlpha, timeElapsed / duration);  // Yavaşça karart
            darkenImage.color = new Color(0, 0, 0, alpha);  // Siyah rengini ve alfa değerini güncelle
            yield return null;
        }

        // Tam olarak karardıktan sonra ölüm işlemini başlat
        Die();
    }
}
