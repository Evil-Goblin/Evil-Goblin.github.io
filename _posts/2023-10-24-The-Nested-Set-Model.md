---
title: The Nested Set Model
author: Evil-Goblin
date: 2023-10-25 18:18:00 +0900
categories: [Jpa, Database]
tags: [Database, RDBMS, NestedSetCollection, Spring, Jpa]
---
# 중첩 세트 모델
- 중첩 세트 모델은 관계형 데이터베이스에서 중첩 집합 컬렉션 ( 트리 또는 계층 ) 을 표현하는 기술이다.
- 계층형 댓글을 구현하는 방법으로서 소개하고자 한다.

![NestedSetModel01](https://github.com/Evil-Goblin/Evil-Goblin.github.io/assets/74400861/14b39005-97cf-4557-8f8e-7d4cd31017b7){: .normal }
- A hierarchy: types of clothing (출처: https://en.wikipedia.org/wiki/Nested_set_model)
- 의류를 계층 구조로 나눈 예시

![NestedSetModel02](https://github.com/Evil-Goblin/Evil-Goblin.github.io/assets/74400861/9d012afa-b729-4ed1-825d-7397f39d515b){: .normal }
- The numbering assigned by tree traversal (출처: https://en.wikipedia.org/wiki/Nested_set_model)
- 트리 순회를 통해 각 노드에 번호를 매긴다.

## 기술 설명
- 각 노드를 두 번 방문하고 방문 순서대로 번호를 할당하는 트리 순회에 따라 노드 번호를 매기는 방식이다.
- 결과 각 노드는 두개의 노드 번호를 부여받게 된다.
- 루트 노드 하나에 너무 많은 자식 노드들이 생성된 경우 중간에 삽입되는 노드를 위해 번호를 다시 매겨야함으로 업데이트 비용이 발생한다.

![NestedSetModel03](https://github.com/Evil-Goblin/Evil-Goblin.github.io/assets/74400861/05dc3a8c-dd38-4df4-a62a-592f623bdac4){: .normal }
- 노드를 추가한 경우 노드에 부여된 노드 번호의 변화 (출처: https://www.werc.cz/blog/2015/07/19/nested-set-model-practical-examples-part-i)
- 새로운 노드르 추가할 때 부모 노드의 `rgt` 값을 기반으로 새로운 노드 번호를 생성하고 해당 값 이상의 모든 노드 번호 값을 갱신해준다.

## 계층형 댓글에 적용
### 댓글 Entity
```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Comment extends BaseEntity {

    public static final int MAX_DEPTH = 5;

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    @Column(name = "comment_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id")
    private Post post;

    @Lob
    private String content;

    @Column(nullable = false)
    private Long depth = 0L;

    @Column(nullable = false)
    private Long leftNode = 1L;

    @Column(nullable = false)
    private Long rightNode = 2L;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "root_comment_id")
    private Comment rootComment;

    @Column(nullable = false)
    private boolean active = true;

    public Comment(Post post, String content) {
        this.post = post;
        this.content = content;
    }

    @Builder
    private Comment(Post post, String content, Long depth, Long leftNode, Long rightNode, Comment rootComment) {
        this.post = post;
        this.content = content;
        this.depth = depth;
        this.leftNode = leftNode;
        this.rightNode = rightNode;
        this.rootComment = rootComment;
    }

    static public Comment newComment(Post post, String content) {
        Comment comment = new Comment(post, content);
        comment.setRootComment(comment);

        return comment;
    }

    public Comment newReplyComment(String content) {
        if (this.depth >= MAX_DEPTH) {
            throw new IllegalStateException("작성 가능한 댓글 깊이를 초과하였습니다.");
        }

        return Comment.builder()
                .post(post)
                .content(content)
                .depth(depth + 1)
                .leftNode(rightNode)
                .rightNode(rightNode + 1)
                .rootComment(rootComment)
                .build();
    }
}
```
- 최상위 레벨의 댓글을 `rootComment` 로 설정하여 `ManyToOne` 연관관계를 만든다.
  - 대댓글들은 최상위 레벨의 댓글을 기준으로만 동작해야 한다.
- 특정 `comment` 에 대댓글을 생성할 경우 새로운 인스턴스를 만들고 업데이트쿼리를 추가로 실행해줘야한다.
  - `Jpa` 는 `bulk update` 전에 영속성 컨텍스트를 `flush` 해버리기 때문에 `insert` 이후 `update` 시 `node` 번호가 중복되게 된다.
  - 인스턴스 생성 -> `update` -> `insert` 순서로 진행되어야 한다.

### 대댓글 저장 Repository Logic
```java
public void saveReplyComment(Comment replyComment) {
    updateNodeNumber(replyComment);

    em.persist(replyComment);
}

private void updateNodeNumber(Comment replyComment) {
    jpaQueryFactory
        .update(comment)
        .set(comment.leftNode, new CaseBuilder()
            .when(leftNodeGOE(replyComment))
            .then(comment.leftNode.add(2))
            .otherwise(comment.leftNode))
        .set(comment.rightNode, comment.rightNode.add(2))
        .where(
            rootCommentEqualTo(replyComment)
                .and(
                    leftNodeGOE(replyComment)
                        .or(rightNodeGOE(replyComment))
                )
        )
        .execute();
}

private BooleanExpression rightNodeGOE(Comment replyComment) {
    return comment.rightNode.goe(replyComment.getLeftNode());
}

private BooleanExpression leftNodeGOE(Comment replyComment) {
    return comment.leftNode.goe(replyComment.getLeftNode());
}

private BooleanExpression rootCommentEqualTo(Comment replyComment) {
    return comment.rootComment.eq(replyComment.getRootComment());
}
```
- 대댓글을 저장하기 위한 노드번호 업데이트
  - `replyComment` 는 `newReplyComment` 에 의해 생성되었기 때문에 노드의 번호는 세팅이 되어있다.
  - 저장하려는 노드의 번호와 같거나 큰 값들을 `+2` 함으로서 저장하려는 노드의 번호가 확보된다.
- 업데이트가 된 이후 `insert` 를 수행한다.

## 테스트 코드 작성시 유의점
- 업데이트 된 내용은 영속성 컨텍스트에 갱신되지 않는다.
  - 업데이트 이후 영속성 컨텍스트를 비워줘야 한다.
- 서비스에서는 영속성 컨텍스트를 잘 컨트롤 하면 된다고 하지만 테스트를 할 경우는 예상치 못한 결과가 발생할 수 있다.

```java
@Autowired
EntityManager em;

private Comment saveReplyAndClear(Comment replyCommentA, String newReplyContent) {
    Comment rereplyCommentA = replyCommentA.newReplyComment(newReplyContent);
    commentRepository.saveReplyComment(rereplyCommentA);

    em.flush();
    em.clear();
    return rereplyCommentA;
}
```
- 테스트를 위해 대댓글 저장 후 영속성 컨텍스트를 비워주는 편의 메소드를 이용한다.
