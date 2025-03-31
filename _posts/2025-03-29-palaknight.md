---
layout: single
title: "[Unity]팔라나이트"
categories: coding
tag: [Unity, C#]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: false
excerpt: "유니티로 만든 팔라독 모방 게임"
---

# 게임 소개

<br/>
한때 큰 인기를 끌었던 추억의 모바일 게임 "팔라독"의 모방 게임을 만들어 보았다.
"팔라독"은 모바일로 출시된 일종의 디펜스 게임으로, 일반적인 타워 디펜스와는 달리, 주인공인 팔라독을 직접 움직이면서 적과 전투할 수 있다는 점이 매력적인 게임이다.

<br/>
![image_1]({{site.url}}/images/2025-03-29-palaknight/image_1.png)

"식량" 자원을 사용하여 아군 유닛을 생성하고, "마나" 자원을 사용하여 주인공의 스킬을 사용할 수 있다. <br/>
적절한 자원 활용을 통해 적들을 해치우고 적의 요새를 부수는 것이 게임의 주 목적이다.

# 게임 구현

## 아군, 적 유닛 구현

모든 아군, 적 유닛은 생성 후 적진으로 달려가다가 공격 사거리 내에 적이 감지되면 공격을 시작한다.
사거리 내 적 감지는 레이케스팅을 활용하여 구현했다.

```c#
//UnitCtrl.cs
private void FixedUpdate()
{
    if (IsDead || GameManager.Instance.IsGameOver)
    {   //게임 오버 상태
        return;
    }
    //유닛의 몸통 가운데 위치에서 움직이는 방향으로 레이케스팅
    RaycastHit2D hit = Physics2D.Raycast
    (transform.position + new Vector3(0, 0.7f, 0), Vector2.right * unitData.moveDir, unitData.range, enemyLayer);
    if (hit.collider != null) //사거리 내에 적이 있는 경우
    {
        if (state != State.Attacking) //공격 실행중이 아닌 경우
        {
            attackCoroutine =   //공격 실행
            StartCoroutine(AttackRoutine(hit.collider.GetComponent<LivingEntity>()));
        }
    }
    else //적이 사거리 내에 없는 경우
    {
        //앞으로 전진
        Move(unitData.speed);
        unitAnimator.SetBool("1_Move", true);
    }
}
```

unitData는 각 유닛의 정보를 담고 있는 스크립터블 오브젝트이다.

```c#
[CreateAssetMenu(menuName = "Scriptable/UnitData", fileName = "Unit Data")]
public class UnitData : ScriptableObject
{
    public UnitCtrl prefab; //유닛의 프리팹
    public int ID;  //고유 번호
    public int cost;    //생성 비용(식량)
    public float damage;    //공격 데미지
    public float hp;    //최대 체력
    public float range;  //공격 사거리
    public float speed;  //이동 속도

    public int moveDir; //이동 방향. 아군이면 1, 적이면 -1
    public float attackSpeed;   //공격 속도
    public float spawnCoolTime; //유닛 생성 쿨타임

    public AudioClip attackClip;    //공격 사운드
    public AudioClip deathClip;     //사망 사운드
}
```

![gif_1]({{site.url}}/images/2025-03-29-palaknight/unitCtrl.gif)

## 적 유닛 생성 알고리즘

적 유닛 생성기 또한 "식량" 자원을 생성하고, 소모하여 유닛을 생성한다. 생성 가능한 유닛의 목록은 "LevelData" 스크립터블 오브젝트에 담아 관리하였다.

```c#
[CreateAssetMenu(menuName = "Scriptable/LevelData", fileName = "LevelData")]
public class LevelData : ScriptableObject
{
    public int worldLv; //월드 레벨
    public int stageLv; //스테이지 레벨
    public int maxFood; //적의 최대 식량
    public int startFood;   //적의 시작시 식량
    public float foodGenTime;   //적의 식량 생성 속도

    public List<UnitData> spawnableUnits;   //생성 가능한 유닛 목록

    public List<int> spawnThresholds;   //각 유닛의 최대 생성 수

    public UnitData[] firstWave;    //첫 번째 웨이브
    public UnitData[] secondWave;   //두 번째 웨이브

}
```

LevelData에는 각 레벨의 난이도 조절을 위한 데이터가 들어있다. 웨이브는 요새의 체력이 60%, 30% 미만으로 떨어질 시 한꺼번에 생성되는 적 유닛들의 목록이다.
<br/>
적이 생성되는 알고리즘은 다음과 같다.

1. 리스트의 첫 번째 유닛부터 생성
2. 생성할 때마다 카운트 증가
3. 카운트가 특정 값(spawnThreshold)에 도달하면 다음 유닛으로 변경
4. 순환하며 마지막 유닛까지 생성한 후 처음부터 반복

```c#
//EnemySpawnCtrl.cs
IEnumerator AutoSpawnEnemy()
{
    while (!GameManager.Instance.IsGameOver)
    {
        yield return new WaitForSeconds(0.1f); // 0.1초마다 생성 시도
        if (spawnCounts[currentUnitIdx] >= spawnThresholds[currentUnitIdx])
        {
            //생성중인 유닛을 다 생성한 경우
            spawnCounts[currentUnitIdx++] = 0;
            if (currentUnitIdx >= spawnableUnits.Count)
            {   //마지막 유닛까지 전부 생성한 경우
                currentUnitIdx = 0;
            }
        }
        else
        {
            if (Food > spawnableUnits[currentUnitIdx].cost) //음식이 충분한 경우
            {
                //유닛 생성 메서드 호출
                CreateUnit(spawnableUnits[currentUnitIdx], false);
                spawnCounts[currentUnitIdx]++;
            }
        }
    }
}
```

## 플레이어 공격 스킬

플레이어는 다양한 무기를 통한 공격이 가능하며, 각 무기마다 소모 기력과 사거리, 데미지가 다르다.

```c#
//playerAttack.cs
public bool WeaponAttack(WeaponData weaponData)
{
    if (Energy < weaponData.energyCost)
    {
        //기력이 충분하지 않은 경우
        return false;
    }
    //플레이어가 바라보는 방향
    Vector2 direction = transform.localScale.x > 0 ? Vector2.right : Vector2.left;
    //direction으로 무기의 사거리 만큼 레이캐스팅
    RaycastHit2D hit = Physics2D.Raycast
    (transform.position + new Vector3(0, 0.5f, 0), direction, weaponData.range, enemyLayer);
    if (hit.collider != null)
    {
        LivingEntity target = hit.collider.GetComponent<LivingEntity>();
        target.OnDamage(weaponData.damage);
    }
    weaponObj.sprite = weaponData.sprite;
    Energy -= weaponData.energyCost;
    UIManager.Instance.UpdateEnergyUI(Energy);
    audioPlayer.PlayOneShot(weaponData.attackClip);
    playerAnimator.SetInteger("WeaponID", weaponData.ID);
    playerAnimator.SetTrigger("2_Attack");
    return true;
}
```

weaponData 또한 스크립터블 오브젝트로, 각 무기의 공격 애니메이션, 스프라이트, 사거리 같은 정보를 담고 있다.

![gif_2]({{site.url}}/images/2025-03-29-palaknight/playerAttack.gif)
플레이어가 사용하는 공격 타입에 따라서 들고 있는 무기, 애니메이션 또한 변경된 것을 볼 수 있다.

# 구현 결과

{% include video id="Qu9QWT4NY7U" provider="youtube" %}
