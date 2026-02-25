# Section 15: Advanced Features

## 🚀 Overview

These optional features extend the base system with intelligent automation, reducing manual triage effort by 60-80%.

---

## 🔁 1. Automatic Duplicate Detection

```python
# Auto-run on every new ticket ingestion
async def auto_detect_duplicate(ticket: Ticket) -> Optional[DuplicateResult]:
    """
    Automatically detect duplicates using high-alpha steering + threshold.
    Runs immediately after embedding is stored.
    """
    # Use duplicate steering vector with high alpha for maximum precision
    steered_q = apply_steering(
        query_embedding=ticket.embedding,
        vectors={"duplicate": 0.7},
    )

    results = await qdrant.search(
        collection_name="tickets",
        query_vector=steered_q.tolist(),
        query_filter=Filter(
            must_not=[FieldCondition(key="ticket_id", match=MatchValue(value=ticket.ticket_id))]
        ),
        limit=3,
        score_threshold=0.92   # High threshold: only flag near-identical tickets
    )

    if results and results[0].score >= 0.92:
        # Auto-link as duplicate
        await db.update_ticket(
            ticket_id=ticket.ticket_id,
            updates={"status": "Duplicate", "duplicate_of": results[0].payload["ticket_id"]}
        )
        await notify_reporter(ticket, f"Marked as duplicate of {results[0].payload['ticket_id']}")
        return DuplicateResult(
            is_duplicate=True,
            original_ticket=results[0].payload["ticket_id"],
            confidence=results[0].score
        )
    return None
```

---

## ⛔ 2. Automatic Rejection Recommendation

```python
async def auto_rejection_recommendation(ticket: Ticket, analysis: AnalysisResult):
    """
    If rejection probability ≥ 80% with high confidence, flag for senior review.
    If rejection probability ≥ 95%, auto-recommend rejection.
    """
    rej_prob = analysis.rejection_prediction.rejection_probability
    confidence = analysis.confidence_score

    if rej_prob >= 95 and confidence >= 80:
        # Auto-recommend rejection (human still confirms)
        await review_queue.add(
            ticket_id=ticket.ticket_id,
            priority="rejection_candidate",
            reason=analysis.rejection_prediction.rejection_reason,
            evidence=analysis.rejection_prediction.similar_rejected_tickets
        )
        await notify_team_lead(
            subject=f"Auto-rejection candidate: {ticket.ticket_id}",
            body=f"AI analysis recommends rejection with {rej_prob}% confidence.\n"
                 f"Reason: {analysis.rejection_prediction.rejection_reason}\n"
                 f"Evidence: {', '.join(analysis.rejection_prediction.similar_rejected_tickets)}"
        )

    elif rej_prob >= 80:
        # Flag for senior engineer review
        await ticket.add_tag("rejection-risk")
        await review_queue.add(ticket_id=ticket.ticket_id, priority="review_needed")
```

---

## 🤖 3. AI Chatbot for Ticket Intelligence

A conversational interface for engineers to query the knowledge base:

```python
# FastAPI WebSocket endpoint for real-time chat
@router.websocket("/ws/chat")
async def ticket_chat(websocket: WebSocket, token: str = Query(...)):
    await websocket.accept()
    session = ChatSession(user=verify_token(token))

    while True:
        message = await websocket.receive_text()

        # Parse user intent
        intent = classify_intent(message)

        if intent == "find_similar":
            description = extract_description(message)
            results = await retrieval_svc.search(description)
            response = format_search_results(results)

        elif intent == "explain_ticket":
            ticket_id = extract_ticket_id(message)
            analysis = await analysis_svc.get_analysis(ticket_id)
            response = format_analysis_for_chat(analysis)

        elif intent == "root_cause_query":
            # General root cause question
            response = await llm_svc.answer_question(
                question=message,
                context=await retrieval_svc.search(message, mode="root_cause")
            )

        else:
            response = await llm_svc.general_answer(message, session.history)

        session.history.append({"user": message, "assistant": response})
        await websocket.send_text(json.dumps({"response": response}))
```

---

## 🔮 4. Root Cause Prediction

```python
# After accumulating 1000+ resolved tickets, train a classifier
# to predict root_cause_category from description alone

class RootCauseClassifier:
    """
    Lightweight classifier for root cause prediction.
    Runs BEFORE LLM to provide fast pre-screening.
    """

    def __init__(self, model_path: str):
        # Fine-tuned sentence transformer + classification head
        self.model = SentenceTransformer(model_path)
        self.classifier = joblib.load(f"{model_path}/classifier.pkl")
        self.label_encoder = joblib.load(f"{model_path}/labels.pkl")

    def predict(self, ticket: TicketInput) -> RootCausePrediction:
        text = format_ticket_for_embedding(ticket)
        embedding = self.model.encode([text], normalize_embeddings=True)
        probs = self.classifier.predict_proba(embedding)[0]
        top_class = self.label_encoder.inverse_transform([probs.argmax()])[0]
        confidence = int(probs.max() * 100)
        return RootCausePrediction(
            category=top_class,
            confidence=confidence,
            probabilities=dict(zip(self.label_encoder.classes_, probs.tolist()))
        )

    @classmethod
    def train(cls, tickets: list[Ticket], output_path: str):
        """Train on resolved tickets with known root causes."""
        from sklearn.linear_model import LogisticRegression
        model = SentenceTransformer("BAAI/bge-large-en-v1.5")
        texts = [format_ticket_for_embedding(t) for t in tickets]
        X = model.encode(texts, normalize_embeddings=True)
        y = [t.root_cause_category for t in tickets]
        clf = LogisticRegression(max_iter=1000, C=10.0).fit(X, y)
        joblib.dump(clf, f"{output_path}/classifier.pkl")
```

---

## 📊 5. Analytics Dashboard Data

```python
# Aggregate queries for dashboard insights

async def get_dashboard_stats(days: int = 30) -> DashboardStats:
    """Key metrics for engineering leadership dashboard."""
    since = datetime.utcnow() - timedelta(days=days)

    return DashboardStats(
        # Volume metrics
        total_tickets=await db.count_tickets(since=since),
        new_tickets=await db.count_tickets(status="New", since=since),
        rejected_rate=await db.rejection_rate(since=since),
        duplicate_rate=await db.duplicate_rate(since=since),

        # AI accuracy metrics
        ai_correct_predictions=await db.ai_accuracy(since=since),
        avg_confidence_score=await db.avg_confidence(since=since),

        # Top root causes
        root_cause_breakdown=await db.root_cause_distribution(since=since),

        # By functional group
        tickets_by_group=await db.group_breakdown(since=since),

        # Resolution time
        avg_resolution_days=await db.avg_resolution_time(since=since),
    )
```

---

> **Next:** [Section 16 — Sample Code Structure](16-sample-code.md)
