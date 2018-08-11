using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerMovement : MonoBehaviour
{

    public float movePower = 1f;
    public float jumpPower = 1f;
    public int maxHealth = 3;

    Rigidbody2D rigid;
    Animator animator;
    private SpriteRenderer renderer;

    Vector3 movement;
    bool isJumping = false;
    bool doubleJump = false;
    bool isDie = false;
    bool isUnBeatTime = false;

    int health = 3;
    int jumpCount = 0;

    // Use this for initialization
    void Start()
    {
        rigid = gameObject.GetComponent<Rigidbody2D>();
        animator = gameObject.GetComponentInChildren<Animator>();
        renderer = gameObject.GetComponentInChildren<SpriteRenderer>();
        health = maxHealth;
    }

    // Update is called once per frame
    void Update()
    {
        if (health == 0)
        {
            if (!isDie)
                Die();
            return;
        }
        if (Input.GetAxisRaw("Horizontal") == 0)
        {
            animator.SetBool("isMoving", false);
        }
        else if (Input.GetAxisRaw("Horizontal") < 0)
        {
            animator.SetBool("isMoving", true);
        }
        else if (Input.GetAxisRaw("Horizontal") > 0)
        {
            animator.SetBool("isMoving", true);
        }

        if (Input.GetButtonDown("Jump") && (!animator.GetBool("isJumping") || (animator.GetBool("isJumping") && doubleJump)))
        {
            jumpCount++;
            isJumping = true;
            animator.SetBool("isJumping", true);
            animator.SetTrigger("doJumping");

            if (jumpCount == 2)
            {
                doubleJump = false;
                jumpCount = 0;
            }
        }
    }

    void FixedUpdate()
    {
        if (health == 0)
            return;

        Move();
        Jump();
    }
    // [movement function]
    void Move()
    {
        Vector3 moveVelocity = Vector3.zero;

        if (Input.GetAxisRaw("Horizontal") < 0)
        {
            moveVelocity = Vector3.left;
            transform.localScale = new Vector3(-1, 1, 1);
        }

        else if (Input.GetAxisRaw("Horizontal") > 0)
        {
            moveVelocity = Vector3.right;
            transform.localScale = new Vector3(1, 1, 1);
        }
        transform.position += moveVelocity * movePower * Time.deltaTime;
    }

    void Jump()
    {
        if (!isJumping)
            return;

        rigid.velocity = Vector2.zero;

        Vector2 jumpVelocity = new Vector2(0, jumpPower);
        rigid.AddForce(jumpVelocity, ForceMode2D.Impulse);

        isJumping = false;
    }

    void Die()
    {
        isDie = true;

        rigid.velocity = Vector2.zero;

        BoxCollider2D[] colls = gameObject.GetComponents<BoxCollider2D>();
        colls[0].enabled = false;
        colls[1].enabled = false;

        Vector2 dieVelocity = new Vector2(0, 20f);
        rigid.AddForce(dieVelocity, ForceMode2D.Impulse);
        Debug.Log("DEad");

        Invoke("RestartStage", 2f);
    }

    void RestartStage()
    {
        GameManager.RestartStage();
    }

    //Attatch Event
    void OnTriggerEnter2D(Collider2D other)
    {
        if ((other.gameObject.layer == 8 || other.gameObject.layer == 9) && rigid.velocity.y < 0)
        {
            animator.SetBool("isJumping", false);//landing
        }

        //block action
        if (other.gameObject.layer == 9 && rigid.velocity.y < 0)
        {
            BlockStatus block = other.gameObject.GetComponent<BlockStatus>();

            switch (block.type)
            {
                case "Up":
                    Vector2 upVelocity = new Vector2(0, block.value);
                    rigid.AddForce(upVelocity, ForceMode2D.Impulse);
                    break;

                case "DoubleJump":
                    jumpCount = 0;
                    doubleJump = true;
                    break;
                case "Portal Enter":
                    Vector3 anotherPortalPos = block.portal.transform.position;
                    Vector3 warpPos = new Vector3(anotherPortalPos.x, anotherPortalPos.y + 2f, anotherPortalPos.z);
                    transform.position = warpPos;
                    break;
            }
        }

        //Creature Kill
        if (other.gameObject.tag == "Creature" && !other.isTrigger && rigid.velocity.y < -10f)
        {
            //Bouncing
            Vector2 killVelocity = new Vector2(0, 15f);
            rigid.AddForce(killVelocity, ForceMode2D.Impulse);

            //Kill Creature
            CreatureMovement creature = other.gameObject.GetComponent<CreatureMovement>();
            creature.Die();

            //Get Score
            ScoreManager.setScore(creature.score);
            Debug.Log("Attatch");
        }

        else if (other.gameObject.tag == "Creature" && !other.isTrigger && !(rigid.velocity.y < -10f)&& !isUnBeatTime)
        {
            Vector2 attackedVelocity = Vector2.zero;
            if (other.gameObject.transform.position.x > transform.position.x)
                attackedVelocity = new Vector2(-5f, 7f);
            else
                attackedVelocity = new Vector2(5f, 7f);
            rigid.AddForce(attackedVelocity, ForceMode2D.Impulse);

            health--;

            //Unbeattime On
            if (health > 0)
            {
                isUnBeatTime = true;
                StartCoroutine("UnBeatTime");
            }
        }

        if (other.gameObject.tag == "Bottom")
        {
            health = 0;
        }

        //get coin
        if (other.gameObject.tag == "Coin" && other.isTrigger)
        {
            //get score
            BlockStatus coin = other.gameObject.GetComponent<BlockStatus>();
            ScoreManager.setScore((int)coin.value);

            Destroy(other.gameObject, 0f);
        }

        if (other.gameObject.tag == "End")
        {
            other.enabled = false;
            GameManager.EndGame();
        }
    }

    void OnGUI()
    {
        GUILayout.BeginArea(new Rect(0, 0, Screen.width, Screen.height));
        GUILayout.BeginHorizontal();
        GUILayout.Space(10);
        GUILayout.BeginVertical();
        GUILayout.Space(15);

        string heart = "";
        for (int i = 0; i < health; i++)
        {
            heart += "★ ";
        }
        GUILayout.Label(heart);

        GUILayout.FlexibleSpace();
        GUILayout.EndVertical();
        GUILayout.FlexibleSpace();
        GUILayout.EndHorizontal();
        GUILayout.EndArea();
    }

    IEnumerator UnBeatTime()
    {
        int countTime = 0;

        SpriteRenderer spr = GetComponent<SpriteRenderer>();

        while (countTime < 10)
        {
            if (countTime % 2 == 0)
            {
                Color color = new Vector4(1, 1, 1, 0.5f);
                transform.GetComponent<Renderer>().material.color = color;
                yield return 0;
                Debug.Log("깜");
            }
            else
            {
                Color color = new Vector4(1, 1, 1, 1f);
                transform.GetComponent<Renderer>().material.color = color;
                yield return 0;
                Debug.Log("빡");
            }
               

            //wait Update frame
            yield return new WaitForSeconds(0.2f);

            countTime++;
        }

        //Alpha effect End
        renderer.color = new Color32(255, 255, 255, 255);

        isUnBeatTime = false;
        yield return null;
    }
}
