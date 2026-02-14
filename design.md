# Design Document: Drishti.ai

## 1. Overview

Drishti.ai is an AI-powered visual trust and fraud detection system that provides real-time image authenticity verification. The system analyzes uploaded images to detect AI-generated content, manipulations, and visual artifacts, providing users with explainable risk assessments.

### Design Philosophy
- **Explainability First**: All detection results must be accompanied by human-readable explanations
- **Real-time Processing**: Optimize for sub-5-second response times
- **Scalable Architecture**: Cloud-native design supporting horizontal scaling
- **Privacy-Centric**: Minimize data retention and ensure secure processing

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌─────────────┐
│   Client    │
│ (Web/API)   │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────┐
│      API Gateway Layer              │
│  - Authentication                   │
│  - Rate Limiting                    │
│  - Request Validation               │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│   Image Processing Service          │
│  - Image Upload Handler             │
│  - Format Validation                │
│  - Preprocessing Pipeline           │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│     Detection Engine                │
│  ┌─────────────────────────────┐   │
│  │ AI Generation Detector      │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │ Manipulation Detector       │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │ Artifact Analyzer           │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │ Metadata Inspector          │   │
│  └─────────────────────────────┘   │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│   Analysis Aggregation Layer        │
│  - Risk Scorer                      │
│  - Explainability Module            │
│  - Report Generator                 │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│      Response Formatter             │
│  - JSON API Response                │
│  - Risk Report Generation           │
└─────────────────────────────────────┘
```

### 2.2 Component Descriptions

#### API Gateway Layer
**Purpose**: Entry point for all client requests
**Responsibilities**:
- Request authentication and authorization
- Rate limiting to prevent abuse
- Input validation and sanitization
- Request routing

**Design Rationale**: Centralized gateway provides security, monitoring, and traffic management in one place.

#### Image Processing Service
**Purpose**: Handle image uploads and preprocessing
**Responsibilities**:
- Accept images in JPEG, PNG formats
- Validate file size and format
- Normalize images for analysis
- Extract and preserve metadata

**Design Rationale**: Separating preprocessing from detection allows independent scaling and easier maintenance.

#### Detection Engine
**Purpose**: Core AI/ML analysis system
**Components**:

1. **AI Generation Detector**
   - Identifies images created by GANs, diffusion models, or other generative AI
   - Uses deep learning models trained on synthetic vs. real image datasets
   - Outputs confidence score (0-1)

2. **Manipulation Detector**
   - Detects copy-move, splicing, and retouching operations
   - Analyzes compression artifacts and noise patterns
   - Identifies inconsistencies in JPEG compression levels

3. **Artifact Analyzer**
   - Lighting inconsistency detection
   - Texture anomaly identification
   - Shadow irregularity analysis
   - Edge artifact detection
   - Color space inconsistencies

4. **Metadata Inspector**
   - EXIF data analysis
   - Software signature detection
   - Timestamp validation
   - GPS data consistency checks

**Design Rationale**: Modular detection components allow independent updates and improvements. Multiple detection methods provide comprehensive coverage and reduce false negatives.

#### Analysis Aggregation Layer
**Purpose**: Combine detection results into actionable insights
**Components**:

1. **Risk Scorer**
   - Aggregates signals from all detectors
   - Applies weighted scoring algorithm
   - Generates risk score (0-100)
   - Classifies into risk categories: Low (0-30), Medium (31-70), High (71-100)

2. **Explainability Module**
   - Generates human-readable explanations
   - Highlights specific artifacts or inconsistencies
   - Provides confidence levels for each finding
   - Supports multiple languages (Phase 1: English, Hindi)

3. **Report Generator**
   - Compiles final Risk_Report
   - Includes visual annotations (optional)
   - Formats output for API or UI consumption

**Design Rationale**: Centralized aggregation ensures consistent scoring logic and enables A/B testing of different scoring algorithms.

---

## 3. Data Models

### 3.1 Image Upload Request
```json
{
  "image": "base64_encoded_string | multipart_file",
  "format": "jpeg | png",
  "metadata_analysis": true,
  "detailed_report": false
}
```

### 3.2 Risk Report Response
```json
{
  "request_id": "uuid",
  "timestamp": "ISO8601",
  "risk_score": 0-100,
  "risk_category": "low | medium | high",
  "classification": {
    "is_ai_generated": {
      "detected": boolean,
      "confidence": 0-1,
      "model_type": "gan | diffusion | unknown"
    },
    "is_manipulated": {
      "detected": boolean,
      "confidence": 0-1,
      "manipulation_types": ["copy-move", "splicing", "retouching"]
    }
  },
  "artifacts_detected": [
    {
      "type": "lighting_inconsistency | texture_anomaly | shadow_irregularity | edge_artifact",
      "severity": "low | medium | high",
      "location": "coordinates or region description",
      "confidence": 0-1
    }
  ],
  "metadata_analysis": {
    "suspicious_patterns": [],
    "software_signatures": [],
    "inconsistencies": []
  },
  "explanation": {
    "summary": "Human-readable summary",
    "details": [
      "Detailed finding 1",
      "Detailed finding 2"
    ],
    "recommendations": "Action recommendations for user"
  },
  "processing_time_ms": integer
}
```

### 3.3 Internal Detection Result
```json
{
  "detector_id": "string",
  "detector_type": "ai_generation | manipulation | artifact | metadata",
  "confidence": 0-1,
  "findings": [],
  "processing_time_ms": integer
}
```

---

## 4. API Design

### 4.1 REST API Endpoints

#### POST /api/v1/analyze
**Purpose**: Analyze uploaded image for authenticity
**Request**:
- Content-Type: multipart/form-data or application/json
- Body: Image file or base64 encoded image
- Optional parameters: detailed_report, metadata_analysis

**Response**: Risk_Report (see Data Models)
**Status Codes**:
- 200: Analysis completed successfully
- 400: Invalid image format or request
- 413: Image file too large
- 429: Rate limit exceeded
- 500: Internal server error

#### GET /api/v1/report/{request_id}
**Purpose**: Retrieve previously generated report
**Response**: Risk_Report
**Status Codes**:
- 200: Report found
- 404: Report not found or expired

#### GET /api/v1/health
**Purpose**: Health check endpoint
**Response**: System status and component health

### 4.2 Rate Limiting
- Free tier: 10 requests per minute
- Authenticated tier: 100 requests per minute
- Enterprise tier: Custom limits

**Design Rationale**: Rate limiting prevents abuse while allowing legitimate use cases. Tiered approach supports different user segments.

---

## 5. AI/ML Model Strategy

### 5.1 AI Generation Detection Model
**Approach**: Deep learning classifier
**Architecture**: CNN-based or Vision Transformer
**Training Data**:
- Real images: Public datasets (COCO, ImageNet subsets)
- Synthetic images: Generated using Stable Diffusion, DALL-E style models, GANs

**Features**:
- Frequency domain analysis
- Noise pattern recognition
- Spectral analysis
- Texture consistency

**Design Rationale**: Combining spatial and frequency domain features improves detection of subtle AI generation artifacts.

### 5.2 Manipulation Detection Model
**Approach**: Error Level Analysis (ELA) + Deep Learning
**Techniques**:
- JPEG compression artifact analysis
- Copy-move detection using SIFT/ORB features
- Splicing detection via boundary analysis
- Noise inconsistency detection

**Design Rationale**: Hybrid approach combining traditional forensic techniques with deep learning provides robust detection across manipulation types.

### 5.3 Artifact Analysis
**Approach**: Rule-based + ML hybrid
**Techniques**:
- Lighting direction estimation
- Shadow consistency checking
- Texture frequency analysis
- Edge sharpness profiling

**Design Rationale**: Some artifacts are better detected with physics-based rules, while others benefit from learned patterns.

### 5.4 Model Updates
- Models deployed as versioned containers
- A/B testing framework for new model versions
- Rollback capability for problematic deployments
- Continuous retraining pipeline with new data

---

## 6. Risk Scoring Algorithm

### 6.1 Scoring Formula
```
risk_score = (
  w1 * ai_generation_confidence +
  w2 * manipulation_confidence +
  w3 * artifact_severity_score +
  w4 * metadata_suspicion_score
) * 100

where:
w1 = 0.35 (AI generation weight)
w2 = 0.35 (Manipulation weight)
w3 = 0.20 (Artifact weight)
w4 = 0.10 (Metadata weight)
```

### 6.2 Risk Categories
- **Low (0-30)**: Image appears authentic with minimal concerns
- **Medium (31-70)**: Some suspicious indicators detected, proceed with caution
- **High (71-100)**: Strong evidence of AI generation or manipulation

**Design Rationale**: Weighted scoring allows tuning based on real-world performance. Weights prioritize direct detection methods over indirect signals.

---

## 7. Explainability Strategy

### 7.1 Explanation Generation
**Approach**: Template-based with dynamic content insertion
**Components**:
1. **Summary Statement**: One-sentence risk assessment
2. **Key Findings**: Bullet points of specific detections
3. **Visual Indicators**: Description of artifacts with locations
4. **Confidence Levels**: Transparency about detection certainty
5. **Recommendations**: Actionable advice for users

### 7.2 Example Explanation
```
Summary: This image shows strong indicators of AI generation.

Key Findings:
- AI generation patterns detected with 87% confidence
- Texture inconsistencies in background region
- Unusual noise distribution typical of diffusion models
- No camera metadata present

Recommendation: Verify the source of this image before trusting its content.
```

**Design Rationale**: Structured explanations provide transparency while remaining accessible to non-technical users.

---

## 8. Technology Stack

### 8.1 Backend
- **Language**: Python 3.10+
- **Framework**: FastAPI (async support, automatic API docs)
- **ML Framework**: PyTorch (model inference)
- **Image Processing**: OpenCV, Pillow, scikit-image
- **API Gateway**: Kong or AWS API Gateway

**Design Rationale**: Python ecosystem provides rich ML/CV libraries. FastAPI offers high performance with async support.

### 8.2 Infrastructure
- **Cloud Provider**: AWS or GCP
- **Compute**: Kubernetes for container orchestration
- **GPU Instances**: For ML inference (NVIDIA T4 or A10G)
- **Storage**: S3-compatible object storage (temporary)
- **Caching**: Redis for rate limiting and result caching
- **Monitoring**: Prometheus + Grafana

**Design Rationale**: Cloud-native architecture enables auto-scaling. Kubernetes provides portability across cloud providers.

### 8.3 ML Model Serving
- **Framework**: TorchServe or TensorFlow Serving
- **Optimization**: ONNX Runtime for inference acceleration
- **Batching**: Dynamic batching for throughput optimization

---

## 9. Performance Optimization

### 9.1 Target Metrics
- **Response Time**: < 5 seconds (p95)
- **Throughput**: 100+ requests/second per instance
- **GPU Utilization**: > 70%

### 9.2 Optimization Strategies
1. **Model Optimization**
   - Quantization (FP16 or INT8)
   - Model pruning for smaller footprint
   - Knowledge distillation for faster inference

2. **Request Processing**
   - Async I/O for non-blocking operations
   - Request batching for GPU efficiency
   - Image preprocessing pipeline optimization

3. **Caching**
   - Cache identical image hashes
   - Cache model outputs for duplicate requests
   - TTL: 1 hour for analysis results

4. **Load Balancing**
   - Horizontal pod autoscaling based on CPU/GPU metrics
   - Request routing to least-loaded instances

**Design Rationale**: Multi-layered optimization ensures meeting performance requirements under varying load conditions.

---

## 10. Security & Privacy

### 10.1 Data Handling
- **Upload Security**: Validate file types, scan for malware
- **Temporary Storage**: Images stored only during processing
- **Retention Policy**: Delete images immediately after analysis (default)
- **Optional Storage**: User can opt-in to store for report retrieval (24-hour TTL)

### 10.2 API Security
- **Authentication**: API key or OAuth 2.0
- **Encryption**: TLS 1.3 for all communications
- **Input Validation**: Strict validation of all inputs
- **Rate Limiting**: Prevent abuse and DDoS

### 10.3 Privacy Compliance
- **Data Minimization**: Process only necessary data
- **No PII Storage**: Avoid storing personal information from images
- **Audit Logging**: Log access and processing events
- **Compliance**: Align with Indian data protection regulations

**Design Rationale**: Privacy-first design builds user trust and ensures regulatory compliance.

---

## 11. Scalability Design

### 11.1 Horizontal Scaling
- **Stateless Services**: All components designed as stateless
- **Auto-scaling**: Based on request queue depth and GPU utilization
- **Load Distribution**: Round-robin with health checks

### 11.2 Capacity Planning
- **Initial Capacity**: 10 GPU instances (1000 req/min)
- **Scaling Triggers**: 
  - Scale up: Queue depth > 50 or GPU util > 80%
  - Scale down: Queue depth < 10 and GPU util < 30%

### 11.3 Database Scaling
- **Metadata Store**: DynamoDB or Cloud Firestore (serverless)
- **Caching Layer**: Redis cluster with replication

**Design Rationale**: Stateless architecture enables seamless scaling. Serverless databases reduce operational overhead.

---

## 12. Monitoring & Observability

### 12.1 Metrics
- **System Metrics**: CPU, memory, GPU utilization
- **Application Metrics**: Request rate, response time, error rate
- **Business Metrics**: Detection accuracy, false positive rate
- **Model Metrics**: Inference time, confidence distributions

### 12.2 Logging
- **Structured Logging**: JSON format with correlation IDs
- **Log Levels**: DEBUG, INFO, WARN, ERROR
- **Retention**: 30 days for application logs

### 12.3 Alerting
- **Critical Alerts**: Service downtime, error rate > 5%
- **Warning Alerts**: Response time > 7s, GPU util > 90%
- **Channels**: PagerDuty, Slack, Email

**Design Rationale**: Comprehensive observability enables proactive issue detection and performance optimization.

---

## 13. Testing Strategy

### 13.1 Unit Testing
- Test individual detector modules
- Test risk scoring algorithm
- Test explanation generation logic
- Target: > 80% code coverage

### 13.2 Integration Testing
- End-to-end API testing
- Test complete analysis pipeline
- Test error handling and edge cases

### 13.3 Model Testing
- Validation on held-out test sets
- Adversarial testing with challenging images
- Performance benchmarking
- A/B testing for model updates

### 13.4 Load Testing
- Simulate peak traffic scenarios
- Test auto-scaling behavior
- Identify performance bottlenecks

**Design Rationale**: Multi-layered testing ensures reliability and performance under real-world conditions.

---

## 14. Deployment Strategy

### 14.1 CI/CD Pipeline
1. **Code Commit**: Trigger automated tests
2. **Build**: Create Docker containers
3. **Test**: Run integration and load tests
4. **Deploy to Staging**: Automated deployment
5. **Validation**: Smoke tests and manual QA
6. **Deploy to Production**: Blue-green deployment
7. **Monitor**: Watch metrics for anomalies

### 14.2 Rollback Strategy
- Keep previous version running during deployment
- Automated rollback on error rate spike
- Manual rollback capability

**Design Rationale**: Automated CI/CD with safety checks enables rapid iteration while maintaining stability.

---

## 15. Future Enhancements

### Phase 2 Features
- Video deepfake detection
- Browser extension for in-browser verification
- WhatsApp integration for viral image checking
- Mobile SDK for app integration

### Phase 3 Features
- Enterprise dashboard with analytics
- Batch processing API
- Custom model training for specific use cases
- Multi-modal analysis (image + text context)

**Design Rationale**: Phased approach allows validating core functionality before expanding scope.

---

## 16. Open Questions & Decisions Needed

1. **Model Selection**: Which specific pre-trained models to use as base?
2. **Cloud Provider**: AWS vs GCP vs Azure?
3. **Pricing Model**: Free tier limits and paid tier pricing?
4. **Language Support**: Priority for regional language support?
5. **Legal Compliance**: Specific certifications or audits required?

---

## 17. Success Metrics

### Technical Metrics
- Detection accuracy > 90% on test set
- False positive rate < 5%
- Response time < 5 seconds (p95)
- System uptime > 99%

### Business Metrics
- User adoption rate
- API integration count
- User satisfaction score
- Fraud prevention impact

**Design Rationale**: Clear success metrics enable objective evaluation of system effectiveness.

---

## 18. Correctness Properties

### Property 1: Response Time Bound
**Description**: All analysis requests must complete within acceptable time limits
**Formal Property**: For all valid image inputs, processing_time_ms < 5000
**Test Strategy**: Property-based testing with varied image sizes and formats

### Property 2: Risk Score Range
**Description**: Risk scores must always be within valid range
**Formal Property**: For all analysis results, 0 <= risk_score <= 100
**Test Strategy**: Property-based testing with diverse detection scenarios

### Property 3: Explanation Completeness
**Description**: Every risk report must include a non-empty explanation
**Formal Property**: For all risk_reports where risk_score > 0, explanation.summary != "" AND len(explanation.details) > 0
**Test Strategy**: Property-based testing across all risk categories

### Property 4: Detection Consistency
**Description**: Identical images should produce identical results
**Formal Property**: For all images I, analyze(I) == analyze(I) when called within cache TTL
**Test Strategy**: Property-based testing with image hash verification

### Property 5: Format Support
**Description**: System must accept all specified image formats
**Formal Property**: For all images in {JPEG, PNG}, upload succeeds OR returns specific format error
**Test Strategy**: Property-based testing with valid and invalid formats

### Property 6: API Response Structure
**Description**: All successful API responses must conform to schema
**Formal Property**: For all successful requests, response validates against Risk_Report schema
**Test Strategy**: Property-based testing with schema validation

---

## 19. Testing Framework

**Framework**: pytest with hypothesis for property-based testing
**Coverage Tool**: pytest-cov
**Load Testing**: Locust
**API Testing**: pytest + httpx

---

## Appendix A: Glossary

- **GAN**: Generative Adversarial Network
- **Diffusion Model**: Type of generative AI model (e.g., Stable Diffusion)
- **ELA**: Error Level Analysis - forensic technique for detecting manipulation
- **EXIF**: Exchangeable Image File Format - metadata standard
- **SIFT/ORB**: Feature detection algorithms for copy-move detection
- **TLS**: Transport Layer Security
- **TTL**: Time To Live

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-15 | Kiro | Initial design document |
