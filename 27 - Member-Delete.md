![Generic badge](https://img.shields.io/badge/JAVA-11-blue.svg) 
![Generic badge](https://img.shields.io/badge/SpringBoot-2.6.3-yellow.svg)
![Generic badge](https://img.shields.io/badge/Gradle-7.4-orange.svg)

***
이번에 구현할 것은 사이트에 가입한 회원을 탈퇴하는 기능을 구현해 보겠습니다.  
일단 로그인이 된 상태여야 하며, 회원 탈퇴하기 전에 본인이 맞는 지 이메일과 비밀번호를 입력하여 확인하는 절차를 가집니다.  
그리고 탈퇴가 성공적으로 이루어지게 되면 본인이 작성한 게시글, 댓글, 추천 등이 같이 삭제가 이루어집니다.  

그러면 먼저 `BoardRepository`, `CommentRepository`, `LikesRepository`에 각각 `Member Entity`에서 `PK값`을 이용하여  
회원이 작성했던 게시물이나 댓글 등을 조회하고 조회한 것을 토대로 삭제하도록 구현하겠습니다.  

## Repository 구현(Board, Comment, Likes)  
각 Repository에 구현할 메서드들은 회원 PK값을 통해 데이터를 조회하는 `findByMemberId()`로 `List<Long>`형태로 각 `Entity`의 `PK 값`을 여러 개의 데이터를 출력하도록 합니다.    
그리고 그렇게 조회한 각 `Entity`의 id값들을 토대로 모든 데이터들을 삭제하는 `deleteAllByIdIn()`를 구현합니다.  
이 메서드들은 `@Query`을 통해 SQL 언어로 작성하며, 각 Repository의 메서드들은 테이블의 이름만 다르고 틀은 똑같습니다.  
<details>
<summary>BoardRepository</summary>

```java
package com.moon.aza.repository;

import com.moon.aza.entity.Board;
import com.moon.aza.repository.search.SearchBoardRepository;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

@Repository
public interface BoardRepository extends JpaRepository<Board, Long>, SearchBoardRepository {
  
  ...생략
  
    @Query("select b.id from Board b where b.member.id = :memberId")
    List<Long> findByMemberId(@Param("memberId") Long memberId);

    @Modifying
    @Transactional
    @Query("delete from Board b where b.id in :ids")
    void deleteAllByIdIn(@Param("ids") List<Long> ids);
}

```
</details>  

<details>
<summary>CommentRepository</summary>

```java
  package com.moon.aza.repository;

import com.moon.aza.entity.Board;
import com.moon.aza.entity.Comment;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;


@Repository
public interface CommentRepository extends JpaRepository<Comment, Long> {
    
  ...생략
  
    @Query("select c.commentNum from Comment c where c.member.id = :memberId")
    List<Long> findByMemberId(@Param("memberId") Long memberId);

    @Modifying
    @Transactional
    @Query("delete from Comment c where c.commentNum in :ids")
    void deleteAllByIdIn(@Param("ids") List<Long> ids);

}

```
</details>

<details>
<summary>Likesepository</summary>

```java
  package com.moon.aza.repository;

import com.moon.aza.entity.Likes;
import com.moon.aza.repository.likes.LikesCustomRepository;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;


@Repository
public interface LikesRepository extends JpaRepository<Likes, Long>, LikesCustomRepository {
    @Query("select l.likeId from Likes l where l.member.id = :memberId")
    List<Long> findByMemberId(@Param("memberId") Long memberId);

    @Modifying
    @Transactional
    @Query("delete from Likes l where l.likeId in :ids")
    void deleteAllByIdIn(@Param("ids") List<Long> ids);
}

```
  
</details>

이제 `Service` 계층에서 회원 탈퇴 기능이 동작하도록 구현합니다.  

## MemberService 구현
```java
  package com.moon.aza.service;

import com.moon.aza.dto.SignUpForm;
import com.moon.aza.entity.Member;
import com.moon.aza.repository.BoardRepository;
import com.moon.aza.repository.CommentRepository;
import com.moon.aza.repository.LikesRepository;
import com.moon.aza.repository.MemberRepository;
import com.moon.aza.support.UserMember;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.validation.Errors;
import org.springframework.validation.FieldError;

import javax.mail.MessagingException;
import javax.mail.internet.MimeMessage;
import java.util.*;

@RequiredArgsConstructor
@Transactional
@Service
public class MemberService implements UserDetailsService {
    private final MemberRepository memberRepository;
    private final BoardRepository boardRepository;
    private final LikesRepository likesRepository;
    private final CommentRepository commentRepository;
    private final JavaMailSender mailSender;
    private final PasswordEncoder passwordEncoder;
    @Value("${spring.mail.username}")
    private String from;

    public void deleteMember(Member member){
        List<Long> boardIds = boardRepository.findByMemberId(member.getId());
        List<Long> likesIds = likesRepository.findByMemberId(member.getId());
        List<Long> commentIds = commentRepository.findByMemberId(member.getId());
        if(!commentIds.isEmpty()){
            commentRepository.deleteAllByIdIn(commentIds);
        }
        if(!likesIds.isEmpty()){
            likesRepository.deleteAllByIdIn(likesIds);
        }
        if(!boardIds.isEmpty()){
            boardRepository.deleteAllByIdIn(boardIds);
        }
        memberRepository.deleteById(member.getId());
    }
  ...
}

```
`deleteMember()` 메서드에는 `Repository` 계층에서 구현한 `findByMemberId()`를 통해 탈퇴할 회원이 작성했던 것들을 모두 조회합니다.  
그리고 `deleteAllByIdIn()` 메서드로 조회된 데이터들을 모두 삭제합니다.  
그 후 Database에서 `MemberRepository`의 `deleteById`를 통해 회원을 삭제하도록 합니다.  

왜 이렇게 삭제하냐면 그냥 회원을 삭제시키면 회원의 `PK값`을 참조하고 있는 테이블들은 존재하지 않는 테이블을 참조하고 있는 셈이므로  
불안정한 상태가 되어 `외래키 제약 조건`에 대한 예외가 발생하게 됩니다. 그렇기 때문에 저같은 경우에는 회원을 삭제하기 전에 이 회원을 참조하고 있는 테이블들을 먼저 삭제하도록 했습니다.  
  
## SettingController 구현
```java
  package com.moon.aza.controller;

import com.moon.aza.dto.NicknameForm;
import com.moon.aza.dto.PasswordForm;
import com.moon.aza.entity.Member;
import com.moon.aza.service.MemberService;
import com.moon.aza.support.CurrentMember;
import com.moon.aza.validator.NicknameFormValidator;
import com.moon.aza.validator.PasswordFormValidator;
import lombok.RequiredArgsConstructor;
import lombok.extern.log4j.Log4j2;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.Errors;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.InitBinder;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

import javax.servlet.http.HttpSession;
import javax.validation.Valid;

@Log4j2
@RequiredArgsConstructor
@Controller
public class SettingController {
    private final NicknameFormValidator nicknameFormValidator;
    private final PasswordFormValidator passwordFormValidator;
    private final PasswordEncoder passwordEncoder;
    private final MemberService memberService;

    // 회원 탈퇴
    @GetMapping("/settings/member-delete")
    public void memberForm(@CurrentMember Member member, Model model){
        log.info("/settings/member-delete");
        model.addAttribute(member);
    }
    // 회원 탈퇴
    @PostMapping("/settings/member-delete")
    public String memberDelete(@CurrentMember Member member, String email, String password,
                               HttpSession httpSession, Model model){
        Member result = memberService.findMemberByEmail(email);
        if(result==null || !passwordEncoder.matches(password, member.getPassword())){
            model.addAttribute("error", "유효하지 않은 정보입니다.");
            return "/settings/member-delete";
        }
        memberService.deleteMember(member);
        httpSession.invalidate();
        return "redirect:/";
    }
  ...생략
}

```
먼저 `GET`방식으로 `/settings/member-delete`로 요청이 오면, 회원 탈퇴 하기 전에 본인 정보가 맞는지 확인하는 `View`를 반환하도록 합니다.  
그리고 `POST` 방식으로 `/settings/member-delete`로 요청이 오면, 회원 탈퇴 동작을 진행합니다.    
먼저 탈퇴할 회원의 정보(email, password) 데이터를 받아 이 회원 존재의 유무를 판별하고 Database에 암호화되어 저장된 패스워드와  
확인을 위해 입력한 패스워드가 맞는지 판별합니다. 이메일 또는 패스워드가 유효하지 않은 정보로 판별이 되면, `error`를 전달하도록 합니다.  

유효하면 `MemberService`의 `deleteMember()` 메서드를 호출하여 실제로 회원 탈퇴 동작을 진행하도록 합니다.  
그 후, `httpSession.invalidate()`를 통해 로그아웃하여 홈 경로인 `/`로 리다이렉트하도록 합니다.  

## 결과
초기 화면(/settings/member-delete)
![1](https://user-images.githubusercontent.com/60730405/167430938-995aa1d0-ecc0-4640-ac72-bf6fd63c1cf2.JPG)

유효하지 않은 회원 정보를 입력 시
![2](https://user-images.githubusercontent.com/60730405/167430215-40e07345-2b15-41f2-8f51-7b7b08fc0aa3.JPG)

성공적으로 회원 탈퇴 시 MySQL Workbench에서 `Member` 테이블 확인
![3](https://user-images.githubusercontent.com/60730405/167430513-90f00bc7-92b5-43c5-856c-5eabfb79d1a8.JPG)

