class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    
    # Pattern A: Django cached_property (instance-level cache)
    @cached_property
    def reading_time(self):
        """
        Expensive calculation cached per instance
        Cleared when instance is deleted from memory
        """
        word_count = len(self.content.split())
        return max(1, word_count // 200)
    
    @cached_property
    def related_posts(self):
        """
        Expensive query cached per instance
        Only runs once per request
        """
        return Post.objects.filter(
            tags__in=self.tags.all()
        ).exclude(id=self.id).distinct()[:5]
    
    # Pattern B: Redis/Memcached cache (persistent)
    def get_like_count(self):
        """
        Cache in Redis with automatic invalidation
        """
        cache_key = f'post:{self.id}:like_count'
        count = cache.get(cache_key)
        
        if count is None:
            count = self.likes.count()
            # Cache for 5 minutes
            cache.set(cache_key, count, 60 * 5)
        
        return count
    
    def invalidate_like_count_cache(self):
        """Call when likes change"""
        cache.delete(f'post:{self.id}:like_count')
    
    # Pattern C: Database-stored computed field
    likes_count = models.IntegerField(default=0, db_index=True)
    comments_count = models.IntegerField(default=0)
    
    def update_counts(self):
        """Update denormalized counts"""
        self.likes_count = self.likes.count()
        self.comments_count = self.comments.count()
        self.save(update_fields=['likes_count', 'comments_count'])


# Signal to auto-update counts
@receiver(post_save, sender=Like)
def update_post_like_count(sender, instance, created, **kwargs):
    """Update like count when like is added"""
    if created:
        Post.objects.filter(id=instance.post_id).update(
            likes_count=F('likes_count') + 1
        )

@receiver(post_delete, sender=Like)
def decrease_post_like_count(sender, instance, **kwargs):
    """Decrease like count when like is deleted"""
    Post.objects.filter(id=instance.post_id).update(
        likes_count=F('likes_count') - 1
    )
```

**Comparison:**

| Pattern | Speed | Persistence | Invalidation | Use Case |
|---------|-------|-------------|--------------|----------|
| `@cached_property` | Fast | Request only | Automatic | Heavy computation, same request |
| Redis cache | Very fast | Cross-request | Manual | Expensive queries, high read |
| DB field | Fast | Permanent | Manual | Frequently accessed, needs indexing |

---

#### Pattern 8: Model Choices (Enterprise Pattern)

```python
from django.db import models

# Pattern A: Simple choices (Django 3.0+)
class Post(models.Model):
    class Status(models.TextChoices):
        DRAFT = 'draft', 'Draft'
        PUBLISHED = 'published', 'Published'
        ARCHIVED = 'archived', 'Archived'
    
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.DRAFT,
        db_index=True
    )
    
    # Use choices in code
    def publish(self):
        self.status = self.Status.PUBLISHED
        self.save()
    
    @property
    def is_published(self):
        return self.status == self.Status.PUBLISHED


# Pattern B: Integer choices with logic
class Priority(models.IntegerChoices):
    LOW = 1, 'Low Priority'
    MEDIUM = 2, 'Medium Priority'
    HIGH = 3, 'High Priority'
    URGENT = 4, 'Urgent'
    
    @classmethod
    def get_color(cls, value):
        """Get color for priority"""
        colors = {
            cls.LOW: 'green',
            cls.MEDIUM: 'yellow',
            cls.HIGH: 'orange',
            cls.URGENT: 'red'
        }
        return colors.get(value, 'gray')


class Task(models.Model):
    title = models.CharField(max_length=200)
    priority = models.IntegerField(
        choices=Priority.choices,
        default=Priority.MEDIUM
    )
    
    @property
    def priority_color(self):
        return Priority.get_color(self.priority)


# Pattern C: Complex choices with metadata
class OrderStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    PROCESSING = 'processing', 'Processing'
    SHIPPED = 'shipped', 'Shipped'
    DELIVERED = 'delivered', 'Delivered'
    CANCELLED = 'cancelled', 'Cancelled'
    
    @classmethod
    def get_transitions(cls, current_status):
        """Valid state transitions"""
        transitions = {
            cls.PENDING: [cls.PROCESSING, cls.CANCELLED],
            cls.PROCESSING: [cls.SHIPPED, cls.CANCELLED],
            cls.SHIPPED: [cls.DELIVERED],
            cls.DELIVERED: [],
            cls.CANCELLED: []
        }
        return transitions.get(current_status, [])
    
    @classmethod
    def can_transition(cls, from_status, to_status):
        """Check if transition is valid"""
        return to_status in cls.get_transitions(from_status)


class Order(models.Model):
    status = models.CharField(
        max_length=20,
        choices=OrderStatus.choices,
        default=OrderStatus.PENDING
    )
    
    def change_status(self, new_status):
        """Change status with validation"""
        if not OrderStatus.can_transition(self.status, new_status):
            raise ValueError(
                f"Cannot transition from {self.status} to {new_status}"
            )
        
        old_status = self.status
        self.status = new_status
        self.save()
        
        # Trigger event
        status_changed.send(
            sender=self.__class__,
            instance=self,
            old_status=old_status,
            new_status=new_status
        )


# Pattern D: Database-backed choices (dynamic)
class Category(models.Model):
    """
    Use when choices need to be managed by users
    """
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(unique=True)
    is_active = models.BooleanField(default=True)
    order = models.IntegerField(default=0)
    
    class Meta:
        ordering = ['order', 'name']
        verbose_name_plural = 'categories'


class Post(models.Model):
    category = models.ForeignKey(
        Category,
        on_delete=models.PROTECT,  # Don't allow deleting if posts exist
        related_name='posts'
    )
    
    # Cache choices for performance
    @staticmethod
    def get_category_choices():
        """Get active categories for forms"""
        cache_key = 'active_categories'
        choices = cache.get(cache_key)
        
        if choices is None:
            choices = list(
                Category.objects.filter(is_active=True)
                .values_list('id', 'name')
            )
            cache.set(cache_key, choices, 60 * 60)  # 1 hour
        
        return choices
```

---

### Deep Dive: Django Signals - When & How to Use

#### Understanding Signal Execution

```python
# What happens when you save a model?

post = Post(title="My Post", content="...")
post.save()

# Execution order:
1. pre_save signal sent
   ↓
2. Model validation (if using full_clean)
   ↓
3. INSERT/UPDATE SQL executed
   ↓
4. post_save signal sent
   ↓
5. Save complete


# Signal execution is SYNCHRONOUS
# This means:
@receiver(post_save, sender=Post)
def slow_signal(sender, instance, **kwargs):
    time.sleep(10)  # This BLOCKS the save() call!
    
post.save()  # Takes 10+ seconds to complete!
```

**Critical principle:** Signals run in the same transaction as the save()

```python
from django.db import transaction

@receiver(post_save, sender=Post)
def signal_in_transaction(sender, instance, **kwargs):
    # This runs INSIDE the transaction
    # If this fails, the save() is rolled back
    external_api_call()  # ⚠️ Dangerous!

# Better approach:
@receiver(post_save, sender=Post)
def signal_async(sender, instance, **kwargs):
    # Delegate to async task
    external_api_call_task.delay(instance.id)
```

---

#### Signal Pattern 1: Audit Logging

```python
from django.contrib.contenttypes.models import ContentType
from django.db.models.signals import pre_save, post_save, post_delete

class AuditLog(models.Model):
    """
    Track all changes to models
    """
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    action = models.CharField(max_length=10)  # CREATE, UPDATE, DELETE
    changes = models.JSONField(null=True)
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    timestamp = models.DateTimeField(auto_now_add=True)
    ip_address = models.GenericIPAddressField(null=True)


# Store current user in thread-local
import threading
_thread_locals = threading.local()

def get_current_user():
    return getattr(_thread_locals, 'user', None)

def set_current_user(user):
    _thread_locals.user = user

# Middleware to set user
class AuditMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        set_current_user(request.user if request.user.is_authenticated else None)
        response = self.get_response(request)
        set_current_user(None)
        return response


# Signal handlers
@receiver(pre_save)
def capture_changes(sender, instance, **kwargs):
    """Capture what changed"""
    if instance.pk:
        try:
            old = sender.objects.get(pk=instance.pk)
            instance._old_values = model_to_dict(old)
        except sender.DoesNotExist:
            instance._old_values = None
    else:
        instance._old_values = None


@receiver(post_save)
def log_save(sender, instance, created, **kwargs):
    """Log create/update"""
    if sender == AuditLog:
        return  # Don't log audit logs
    
    changes = None
    if not created and hasattr(instance, '_old_values'):
        new_values = model_to_dict(instance)
        changes = {
            field: {
                'old': instance._old_values.get(field),
                'new': new_values.get(field)
            }
            for field in new_values
            if instance._old_values.get(field) != new_values.get(field)
        }
    
    AuditLog.objects.create(
        content_type=ContentType.objects.get_for_model(sender),
        object_id=instance.pk,
        action='CREATE' if created else 'UPDATE',
        changes=changes,
        user=get_current_user()
    )


@receiver(post_delete)
def log_delete(sender, instance, **kwargs):
    """Log deletion"""
    if sender == AuditLog:
        return
    
    AuditLog.objects.create(
        content_type=ContentType.objects.get_for_model(sender),
        object_id=instance.pk,
        action='DELETE',
        changes=model_to_dict(instance),
        user=get_current_user()
    )
```

**When to use audit logging:**
- Financial systems
- Healthcare (HIPAA compliance)
- Government systems
- Any regulated industry

---

#### Signal Pattern 2: Cache Invalidation

```python
from django.core.cache import cache
from django.db.models.signals import post_save, post_delete, m2m_changed

@receiver(post_save, sender=Post)
def invalidate_post_cache(sender, instance, **kwargs):
    """Invalidate caches when post changes"""
    # Invalidate single post cache
    cache.delete(f'post:{instance.id}')
    
    # Invalidate list caches
    cache.delete('post_list:all')
    cache.delete(f'post_list:author:{instance.author_id}')
    
    # Invalidate related caches
    if instance.category_id:
        cache.delete(f'post_list:category:{instance.category_id}')


@receiver(post_delete, sender=Post)
def invalidate_on_delete(sender, instance, **kwargs):
    """Invalidate caches when post is deleted"""
    invalidate_post_cache(sender, instance)


@receiver(m2m_changed, sender=Post.tags.through)
def invalidate_on_tag_change(sender, instance, **kwargs):
    """Invalidate when tags change"""
    cache.delete(f'post:{instance.id}:tags')
    
    # Invalidate tag-based lists
    for tag in instance.tags.all():
        cache.delete(f'post_list:tag:{tag.id}')


# Pattern: Automatic cache key generation
class CacheInvalidationMixin:
    """
    Mixin to automatically generate cache keys
    """
    
    @classmethod
    def get_cache_keys(cls, instance):
        """Override to define cache keys"""
        return [
            f'{cls.__name__.lower()}:{instance.id}',
            f'{cls.__name__.lower()}_list:all',
        ]
    
    @classmethod
    def invalidate_cache(cls, instance):
        """Invalidate all cache keys"""
        keys = cls.get_cache_keys(instance)
        cache.delete_many(keys)


class Post(CacheInvalidationMixin, models.Model):
    # ... fields ...
    
    @classmethod
    def get_cache_keys(cls, instance):
        keys = super().get_cache_keys(instance)
        keys.extend([
            f'post_list:author:{instance.author_id}',
            f'post_list:category:{instance.category_id}',
        ])
        return keys


@receiver(post_save)
@receiver(post_delete)
def auto_invalidate_cache(sender, instance, **kwargs):
    """Automatically invalidate cache for any model with mixin"""
    if isinstance(instance, CacheInvalidationMixin):
        sender.invalidate_cache(instance)
```

---

#### Signal Pattern 3: Search Index Updates

```python
from django.db.models.signals import post_save, post_delete
from elasticsearch import Elasticsearch

es = Elasticsearch(['localhost:9200'])

@receiver(post_save, sender=Post)
def update_search_index(sender, instance, created, **kwargs):
    """
    Update Elasticsearch index when post changes
    """
    document = {
        'title': instance.title,
        'content': instance.content,
        'author': instance.author.username,
        'created_at': instance.created_at,
        'tags': [tag.name for tag in instance.tags.all()],
    }
    
    # Index or update document
    es.index(
        index='posts',
        id=instance.id,
        body=document
    )


@receiver(post_delete, sender=Post)
def remove_from_search_index(sender, instance, **kwargs):
    """Remove from Elasticsearch when deleted"""
    try:
        es.delete(index='posts', id=instance.id)
    except Exception:
        pass  # Document might not exist


# Better: Async with Celery
from celery import shared_task

@shared_task
def update_search_index_task(post_id):
    """Async search index update"""
    try:
        post = Post.objects.get(id=post_id)
        document = {...}
        es.index(index='posts', id=post.id, body=document)
    except Post.DoesNotExist:
        pass

@receiver(post_save, sender=Post)
def queue_search_index_update(sender, instance, **kwargs):
    """Queue async task"""
    transaction.on_commit(lambda: update_search_index_task.delay(instance.id))
```

**Key insight:** Use `transaction.on_commit()` to ensure task runs AFTER transaction commits

---

#### Signal Pattern 4: Real-time Notifications

```python
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync

@receiver(post_save, sender=Comment)
def notify_post_author(sender, instance, created, **kwargs):
    """
    Notify post author when comment is added
    Real-time via WebSocket
    """
    if not created:
        return
    
    post = instance.post
    if post.author != instance.author:
        # Create notification
        Notification.objects.create(
            user=post.author,
            type='comment',
            message=f'{instance.author.username} commented on your post',
            link=f'/posts/{post.id}/'
        )
        
        # Send WebSocket notification
        channel_layer = get_channel_layer()
        async_to_sync(channel_layer.group_send)(
            f'user_{post.author.id}',
            {
                'type': 'notification',
                'message': f'New comment on "{post.title}"',
                'count': post.author.notifications.unread().count()
            }
        )


@receiver(post_save, sender=Like)
def notify_like(sender, instance, created, **kwargs):
    """Notify when post is liked"""
    if not created:
        return
    
    post = instance.post
    
    # Batch notifications: Only notify for every 10th like
    like_count = post.likes.count()
    if like_count % 10 == 0:
        Notification.objects.create(
            user=post.author,
            type='milestone',
            message=f'Your post reached {like_count} likes! 🎉'
        )
```

---

#### Signal Anti-Patterns (What NOT to do)

```python
# ❌ ANTI-PATTERN 1: Heavy computation in signals
@receiver(post_save, sender=Post)
def bad_signal(sender, instance, **kwargs):
    # This blocks the save() call!
    result = expensive_ml_computation(instance.content)
    instance.ml_score = result
    instance.save()  # ⚠️ This triggers the signal AGAIN!

# ✅ BETTER: Use Celery task
@receiver(post_save, sender=Post)
def good_signal(sender, instance, **kwargs):
    transaction.on_commit(
        lambda: compute_ml_score_task.delay(instance.id)
    )


# ❌ ANTI-PATTERN 2: Recursive saves
@receiver(post_save, sender=Post)
def bad_recursive(sender, instance, **kwargs):
    instance.view_count += 1
    instance.save()  # Triggers signal again → infinite loop!

# ✅ BETTER: Use update() or update_fields
@receiver(post_save, sender=Post)
def good_update(sender, instance, created, **kwargs):
    if created:
        # Use update() - doesn't trigger signals
        Post.objects.filter(id=instance.id).update(view_count=0)
        
        # OR use update_fields
        instance.view_count = 0
        instance.save(update_fields=['view_count'])


# ❌ ANTI-PATTERN 3: Complex business logic
@receiver(post_save, sender=Order)
def bad_complex_logic(sender, instance, **kwargs):
    # 100 lines of order processing logic
    # Payment processing
    # Inventory updates
    # Email sending
    # etc...

# ✅ BETTER: Use service layer
@receiver(post_save, sender=Order)
def good_simple_signal(sender, instance, created, **kwargs):
    if created:
        # Delegate to service
        OrderService.process_new_order(instance.id)


# ❌ ANTI-PATTERN 4: Tight coupling
@receiver(post_save, sender=Post)
def bad_coupling(sender, instance, **kwargs):
    # Directly depends on other apps
    from other_app.models import SomeModel
    from another_app.tasks import some_task
    
    SomeModel.objects.create(...)
    some_task.delay(...)

# ✅ BETTER: Use custom signals
post_published = Signal()

@receiver(post_save, sender=Post)
def check_if_published(sender, instance, **kwargs):
    if instance.status == 'published':
        post_published.send(sender=sender, post=instance)

# Other apps listen to custom signal
@receiver(post_published)
def handle_in_other_app(sender, post, **kwargs):
    # Handle in other app
    pass
```

---

### Deep Thinking: Philosophical Questions & Answers

#### Question 1: "Why use ViewSets instead of APIView?"

**First Principles Analysis:**

1. **What is REST?**
   - Resource-oriented (nouns, not verbs)
   - Standard operations (CRUD)
   - Uniform interface

2. **What is a ViewSet?**
   - Groups related views for a resource
   - Provides standard CRUD by default
   - Works with routers for URL generation

**Answer:**
- **Use ViewSets** when your API follows REST principles (resource-based CRUD)
- **Use APIView** when you have custom operations that don't map to CRUD

**Trade-offs:**
```python
# ViewSet: Less code, more "magic"
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
# 3 lines → full CRUD

# APIView: More code, explicit control
class PostList(APIView):
    def get(self, request):
        # 20 lines of explicit logic
        pass
    
    def post(self, request):
        # 20 lines of explicit logic
        pass
```

**Enterprise decision:** Start with ViewSets, use APIView when you need custom behavior.

---

#### Question 2: "Should I use signals or override save()?"

**Deep thinking:**

Signals are **event-driven** → Decoupled, but hidden behavior
Override save() is **imperative** → Clear, but coupled

```python
# Pattern A: Override save()
class Post(models.Model):
    def save(self, *args, **kwargs):
        # Clear: This code runs on every save
        if not self.slug:
            self.slug = slugify(self.title)
        
        super().save(*args, **kwargs)
        
        # Update search index
        update_search_index(self)

# Pattern B: Signals
class Post(models.Model):
    pass

@receiver(pre_save, sender=Post)
def auto_slug(sender, instance, **kwargs):
    if not instance.slug:
        instance.slug = slugify(instance.title)

@receiver(post_save, sender=Post)
def update_index(sender, instance, **kwargs):
    update_search_index(instance)
```

**Answer:**

| Use save() override when: | Use signals when: |
|---------------------------|-------------------|
| Core model behavior | Side effects / external systems |
| Always happens | Optional / pluggable behavior |
| Single app | Cross-app communication |
| Need to understand flow | Want decoupling |

**Enterprise pattern:**
```python
class Post(models.Model):
    def save(self, *args, **kwargs):
        # Core business logic here
        if not self.slug:
            self.slug = slugify(self.title)
        
        super().save(*args, **kwargs)

# Signals for external concerns
@receiver(post_save, sender=Post)
def external_actions(sender, instance, **kwargs):
    # Cache, notifications, analytics
    pass
```

---

#### Question 3: "When should I denormalize data?"

**First principles:**

**Normalization:** No data duplication → Integrity, but slower reads
**Denormalization:** Duplicate data → Fast reads, but harder updates

```python
# Normalized (standard)
class Post(models.Model):
    title = models.CharField(max_length=200)

class Like(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    user = models.ForeignKey(User, on_delete=models.CASCADE)

# To get like count:
post.likes.count()  # Query every time


# Denormalized
class Post(models.Model):
    title = models.CharField(max_length=200)
    likes_count = models.IntegerField(default=0)  # Duplicated!

# To get like count:
post.likes_count  # No query!
```

**Answer:** Denormalize when:
1. Read >> Write (90%+ reads)
2. Expensive to compute
3. Need for sorting/filtering
4. Can handle eventual consistency

**Enterprise pattern:**
```python
class Post(models.Model):
    # Denormalized for performance
    likes_count = models.IntegerField(default=0, db_index=True)
    comments_count = models.IntegerField(default=0)
    
    # Can still verify
    def verify_counts(self):
        """Check if denormalized counts match reality"""
        actual_likes = self.likes.count()
        actual_comments = self.comments.count()
        
        if self.likes_count != actual_likes or self.comments_count != actual_comments:
            self.likes_count = actual_likes
            self.comments_count = actual_comments
            self.save(update_fields=['likes_count', 'comments_count'])


# Periodic task to fix inconsistencies
@shared_task
def fix_denormalized_counts():
    for post in Post.objects.all():
        post.verify_counts()
```

---

#### Question 4: "How many serializers should one model have?"

**Deep thinking:**

One serializer = Simple, but inflexible
Many serializers = Flexible, but complex

**Answer:** Based on contexts:

```python
# 1. List view (minimal data)
class PostListSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'author_name', 'created_at']

# 2. Detail view (complete data)
class PostDetailSerializer(serializers.ModelSerializer):
    author = UserSerializer()
    comments = CommentSerializer(many=True)
    class Meta:
        model = Post
        fields = '__all__'

# 3. Create (writable fields only)
class PostCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['title', 'content']

# 4. Update (different validation)
class PostUpdateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['title', 'content', 'published']

# 5. Public API (hide sensitive data)
class PostPublicSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        exclude = ['internal_notes', 'draft_content']
```

**Rule of thumb:** 
- Small projects: 1-2 serializers per model
- Medium projects: 3-4 (list, detail, create, update)
- Large projects: 5+ (add public, internal, admin versions)

---

## Conclusion: Enterprise Django REST Patterns

### The Mental Model

```
URLs (Router)          → "How do clients reach resources?"
    ↓
Views (ViewSets)       → "What operations are available?"
    ↓
Serializers            → "How is data validated and formatted?"
    ↓
Models                 → "How is data structured and stored?"
    ↓
Signals                → "What happens after data changes?"
```

### Decision Framework

**Start simple:**
```python
# This is enough for 80% of cases
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
```

**Add complexity when needed:**
- Performance issues → Custom managers, caching, denormalization
- Security needs → Custom permissions, field-level access
- Complex logic → Service layer, custom methods
- Scale requirements → Async views, multi-level caching

### Remember:
1. **Measure before optimizing**
2. **Simple beats clever**
3. **Explicit beats implicit**
4. **Performance matters, but correctness matters more**
5. **Code is read more than written**

---

## Additional Resources

- **Django Docs**: https://docs.djangoproject.com/
- **DRF Docs**: https://www.django-rest-framework.org/
- **Classy DRF**: http://www.cdrf.co/ (Browse DRF class hierarchy)
- **Django Packages**: https://djangopackages.org/
- **Real-world examples**: https://github.com/topics/django-rest-framework

---

**Final wisdom:** 

> "Make it work, make it right, make it fast" - Kent Beck

Start with working code, refactor for clarity, optimize when measured performance requires it.        # Business rule validation
        if data.get('category'):
            category = data['category']
            # Check if user can post in this category
            request = self.context.get('request')
            if request and not category.can_post(request.user):
                raise serializers.ValidationError({
                    'category': 'You do not have permission to post in this category'
                })
        
        return data
    
    # Pattern C: Custom validators
    def validate_content(self, value):
        """Use custom validator functions"""
        # Check minimum word count
        word_count = len(value.split())
        if word_count < 100:
            raise serializers.ValidationError(
                f"Content must be at least 100 words (current: {word_count})"
            )
        
        # Check for spam patterns
        spam_score = calculate_spam_score(value)
        if spam_score > 0.7:
            raise serializers.ValidationError(
                "Content appears to be spam"
            )
        
        return value


# Pattern D: Reusable validators
from rest_framework.validators import UniqueValidator, ValidationError

def validate_no_special_chars(value):
    """Reusable validator function"""
    import re
    if not re.match(r'^[a-zA-Z0-9\s]+#### Pattern 7: Cached Views (Performance)

```python
from django.core.cache import cache
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from rest_framework import viewsets
from rest_framework.response import Response
import hashlib
import json

class CachedPostViewSet(viewsets.ModelViewSet):
    """
    Enterprise caching patterns for high-traffic APIs
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    
    # Pattern A: Simple time-based caching
    @method_decorator(cache_page(60 * 15))  # 15 minutes
    def list(self, request):
        """
        Cache entire response for 15 minutes
        Good for: Public data, infrequent changes
        """
        return super().list(request)
    
    # Pattern B: Custom cache key with query params
    def retrieve(self, request, pk=None):
        """
        Cache with custom key including query parameters
        """
        # Create cache key from URL + query params
        query_string = request.GET.urlencode()
        cache_key = f'post:{pk}:{hashlib.md5(query_string.encode()).hexdigest()}'
        
        # Try cache first
        cached_data = cache.get(cache_key)
        if cached_data:
            return Response(cached_data)
        
        # Cache miss - fetch from database
        response = super().retrieve(request, pk=pk)
        
        # Cache for 1 hour
        cache.set(cache_key, response.data, 60 * 60)
        
        return response
    
    # Pattern C: Cache invalidation on update
    def update(self, request, pk=None):
        """Invalidate cache when data changes"""
        # Perform update
        response = super().update(request, pk=pk)
        
        # Invalidate related caches
        self._invalidate_post_cache(pk)
        
        return response
    
    def destroy(self, request, pk=None):
        """Invalidate cache when deleted"""
        response = super().destroy(request, pk=pk)
        self._invalidate_post_cache(pk)
        return response
    
    def _invalidate_post_cache(self, post_id):
        """Helper to invalidate all cached versions of a post"""
        # Delete specific post cache
        cache.delete(f'post:{post_id}')
        
        # Delete list cache (contains this post)
        cache.delete('post_list')
        
        # Delete user's posts cache
        post = Post.objects.get(id=post_id)
        cache.delete(f'user:{post.author_id}:posts')


# Pattern D: Smart caching with ETags
class SmartCachedViewSet(viewsets.ModelViewSet):
    """
    Use ETags for conditional requests
    Client sends If-None-Match, server returns 304 if unchanged
    """
    
    def retrieve(self, request, pk=None):
        post = self.get_object()
        
        # Generate ETag from updated_at timestamp
        etag = f'"{post.updated_at.timestamp()}"'
        
        # Check If-None-Match header
        if request.META.get('HTTP_IF_NONE_MATCH') == etag:
            return Response(status=304)  # Not Modified
        
        serializer = self.get_serializer(post)
        response = Response(serializer.data)
        response['ETag'] = etag
        return response


# Pattern E: Multi-level caching (Redis + Memory)
from django.core.cache import caches

class MultiLevelCacheViewSet(viewsets.ModelViewSet):
    """
    L1 Cache: In-memory (fastest, small)
    L2 Cache: Redis (fast, larger)
    L3 Cache: Database (slowest, complete)
    """
    
    def retrieve(self, request, pk=None):
        cache_key = f'post:{pk}'
        
        # Try L1 (in-memory)
        local_cache = caches['local']
        data = local_cache.get(cache_key)
        if data:
            return Response(data)
        
        # Try L2 (Redis)
        redis_cache = caches['default']
        data = redis_cache.get(cache_key)
        if data:
            # Populate L1 for next request
            local_cache.set(cache_key, data, 60)
            return Response(data)
        
        # L3: Fetch from database
        post = self.get_object()
        serializer = self.get_serializer(post)
        data = serializer.data
        
        # Populate caches
        redis_cache.set(cache_key, data, 60 * 60)  # 1 hour
        local_cache.set(cache_key, data, 60)       # 1 minute
        
        return Response(data)
```

**Cache configuration:**

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'TIMEOUT': 3600,  # 1 hour default
    },
    'local': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': 'unique-snowflake',
        'TIMEOUT': 60,  # 1 minute
    }
}
```

**When to use caching:**
- **High traffic**: Same data requested frequently
- **Expensive queries**: Complex aggregations, joins
- **External APIs**: Rate-limited third-party calls
- **Read-heavy**: 90%+ reads, few writes

---

#### Pattern 8: Real-time Views (WebSocket Integration)

```python
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync
from rest_framework import viewsets
from rest_framework.decorators import action

class RealtimePostViewSet(viewsets.ModelViewSet):
    """
    Integrate with Django Channels for real-time updates
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    
    def create(self, request):
        """Create post and broadcast to WebSocket clients"""
        response = super().create(request)
        
        # Broadcast to WebSocket
        channel_layer = get_channel_layer()
        async_to_sync(channel_layer.group_send)(
            'posts',  # Group name
            {
                'type': 'post.created',
                'post': response.data
            }
        )
        
        return response
    
    def update(self, request, pk=None):
        """Update post and notify subscribers"""
        response = super().update(request, pk=pk)
        
        channel_layer = get_channel_layer()
        async_to_sync(channel_layer.group_send)(
            f'post.{pk}',  # Specific post channel
            {
                'type': 'post.updated',
                'post': response.data
            }
        )
        
        return response
    
    @action(detail=True, methods=['post'])
    def like(self, request, pk=None):
        """Real-time like counter updates"""
        post = self.get_object()
        Like.objects.get_or_create(user=request.user, post=post)
        
        # Get updated like count
        like_count = post.likes.count()
        
        # Broadcast to all viewing this post
        channel_layer = get_channel_layer()
        async_to_sync(channel_layer.group_send)(
            f'post.{pk}',
            {
                'type': 'post.like_count_updated',
                'post_id': pk,
                'like_count': like_count
            }
        )
        
        return Response({'like_count': like_count})


# WebSocket Consumer (consumers.py)
from channels.generic.websocket import AsyncJsonWebsocketConsumer

class PostConsumer(AsyncJsonWebsocketConsumer):
    async def connect(self):
        self.post_id = self.scope['url_route']['kwargs']['post_id']
        self.room_group_name = f'post.{self.post_id}'
        
        # Join room group
        await self.channel_layer.group_add(
            self.room_group_name,
            self.channel_name
        )
        
        await self.accept()
    
    async def disconnect(self, close_code):
        # Leave room group
        await self.channel_layer.group_discard(
            self.room_group_name,
            self.channel_name
        )
    
    # Receive message from WebSocket
    async def receive_json(self, content):
        pass
    
    # Receive message from room group
    async def post_updated(self, event):
        # Send message to WebSocket
        await self.send_json({
            'type': 'post_updated',
            'post': event['post']
        })
    
    async def post_like_count_updated(self, event):
        await self.send_json({
            'type': 'like_count_updated',
            'post_id': event['post_id'],
            'like_count': event['like_count']
        })
```

**When to use real-time:**
- **Collaborative editing**: Multiple users editing same document
- **Live updates**: Comments, likes, status changes
- **Notifications**: Push notifications to users
- **Dashboards**: Real-time analytics, monitoring

---

### Deep Dive: Serializer Patterns - 10 Different Approaches

#### Pattern 1: Basic ModelSerializer

```python
class PostSerializer(serializers.ModelSerializer):
    """
    Most common pattern: Auto-generate fields from model
    
    Execution:
    1. Introspects Post model
    2. Creates fields based on model fields
    3. Handles validation automatically
    """
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'created_at']
        read_only_fields = ['id', 'created_at']
        
    # Automatic features:
    # - Field types inferred from model
    # - Validators from model constraints
    # - Default values from model
    # - Relationships handled automatically
```

**When to use:**
- **Standard CRUD**: Straightforward model representation
- **Quick prototypes**: Fast development
- **80% of cases**: Sufficient for most needs

---

#### Pattern 2: Explicit Field Declaration

```python
class PostSerializer(serializers.Serializer):
    """
    Manual field declaration - full control
    More verbose but explicit
    """
    
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    content = serializers.CharField(style={'base_template': 'textarea.html'})
    author_id = serializers.IntegerField()
    published = serializers.BooleanField(default=False)
    created_at = serializers.DateTimeField(read_only=True)
    
    def validate_title(self, value):
        """Field-level validation"""
        if len(value) < 5:
            raise serializers.ValidationError("Title too short")
        return value
    
    def validate(self, data):
        """Object-level validation"""
        if data.get('published') and not data.get('content'):
            raise serializers.ValidationError(
                "Cannot publish post without content"
            )
        return data
    
    def create(self, validated_data):
        """Must implement create manually"""
        return Post.objects.create(**validated_data)
    
    def update(self, instance, validated_data):
        """Must implement update manually"""
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.published = validated_data.get('published', instance.published)
        instance.save()
        return instance
```

**When to use:**
- **Non-model data**: Data not backed by Django models
- **Custom logic**: Need full control over serialization
- **External APIs**: Serializing third-party API responses
- **Learning**: Understanding how serializers work

---

#### Pattern 3: Nested Serializers (Read-only)

```python
class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'profile_picture']

class CommentSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    
    class Meta:
        model = Comment
        fields = ['id', 'content', 'author', 'created_at']

class PostDetailSerializer(serializers.ModelSerializer):
    """
    Nested serializers for rich representation
    READ-ONLY: Can't create/update nested objects
    """
    author = AuthorSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    tags = serializers.StringRelatedField(many=True, read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'comments', 'tags', 'created_at']

# Output:
{
    "id": 1,
    "title": "My Post",
    "content": "...",
    "author": {
        "id": 5,
        "username": "john",
        "email": "john@example.com",
        "profile_picture": "https://..."
    },
    "comments": [
        {
            "id": 10,
            "content": "Great post!",
            "author": {"id": 6, "username": "jane", ...},
            "created_at": "2025-10-06T10:00:00Z"
        }
    ],
    "tags": ["django", "python", "api"]
}
```

**When to use:**
- **Detail views**: Rich representation with related data
- **Read-only**: Displaying data, not accepting input
- **API clients**: Reduce number of requests needed

**Performance consideration:**

```python
# ⚠️ WARNING: This causes N+1 queries!
class PostDetailSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)  # Query for each post!
    comments = CommentSerializer(many=True, read_only=True)  # N queries!

# ✅ SOLUTION: Optimize queryset in view
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        queryset = super().get_queryset()
        
        if self.action == 'retrieve':
            queryset = queryset.select_related('author').prefetch_related(
                'comments', 'comments__author', 'tags'
            )
        
        return queryset
```

---

#### Pattern 4: Writable Nested Serializers

```python
class CommentCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Comment
        fields = ['content', 'author_id']

class PostCreateSerializer(serializers.ModelSerializer):
    """
    Accept nested data for creation
    More complex: must handle creation of related objects
    """
    comments = CommentCreateSerializer(many=True, required=False)
    tags = serializers.ListField(
        child=serializers.CharField(max_length=50),
        required=False
    )
    
    class Meta:
        model = Post
        fields = ['title', 'content', 'comments', 'tags']
    
    def create(self, validated_data):
        """Handle nested creation"""
        comments_data = validated_data.pop('comments', [])
        tags_data = validated_data.pop('tags', [])
        
        # Create post
        post = Post.objects.create(**validated_data)
        
        # Create comments
        for comment_data in comments_data:
            Comment.objects.create(post=post, **comment_data)
        
        # Create/get tags
        for tag_name in tags_data:
            tag, created = Tag.objects.get_or_create(name=tag_name)
            post.tags.add(tag)
        
        return post
    
    def update(self, instance, validated_data):
        """Handle nested updates"""
        comments_data = validated_data.pop('comments', None)
        tags_data = validated_data.pop('tags', None)
        
        # Update post fields
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.save()
        
        # Update comments if provided
        if comments_data is not None:
            # Delete existing comments
            instance.comments.all().delete()
            
            # Create new comments
            for comment_data in comments_data:
                Comment.objects.create(post=instance, **comment_data)
        
        # Update tags if provided
        if tags_data is not None:
            instance.tags.clear()
            for tag_name in tags_data:
                tag, created = Tag.objects.get_or_create(name=tag_name)
                instance.tags.add(tag)
        
        return instance


# Request example:
POST /api/posts/
{
    "title": "My Post",
    "content": "Post content",
    "comments": [
        {"content": "First comment", "author_id": 5},
        {"content": "Second comment", "author_id": 6}
    ],
    "tags": ["django", "python"]
}
```

**When to use:**
- **Complex creation**: Create post with comments in one request
- **Atomic operations**: Related objects created together
- **Better UX**: Single API call instead of multiple

**Trade-offs:**
- ✅ **Pros**: Fewer API calls, atomic, better UX
- ❌ **Cons**: Complex code, harder validation, error handling

---

#### Pattern 5: Dynamic Field Serializers

```python
class DynamicFieldsModelSerializer(serializers.ModelSerializer):
    """
    A ModelSerializer that takes an additional `fields` argument
    to dynamically include/exclude fields
    
    Usage: GET /api/posts/?fields=id,title,author
    """
    
    def __init__(self, *args, **kwargs):
        # Extract fields from context or kwargs
        fields = kwargs.pop('fields', None)
        
        # Optionally get fields from request
        request = kwargs.get('context', {}).get('request')
        if request and not fields:
            fields = request.query_params.get('fields')
        
        super().__init__(*args, **kwargs)
        
        if fields:
            fields = fields.split(',') if isinstance(fields, str) else fields
            # Drop any fields not specified
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)


class PostSerializer(DynamicFieldsModelSerializer):
    author = UserSerializer(read_only=True)
    comments_count = serializers.IntegerField(read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'comments_count', 'created_at']


# Usage in view:
class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    
    def get_serializer(self, *args, **kwargs):
        """Pass fields from query params to serializer"""
        serializer_class = self.get_serializer_class()
        kwargs['context'] = self.get_serializer_context()
        return serializer_class(*args, **kwargs)


# API Calls:
GET /api/posts/?fields=id,title
# Response: [{"id": 1, "title": "Post 1"}, ...]

GET /api/posts/?fields=id,title,author
# Response: [{"id": 1, "title": "Post 1", "author": {...}}, ...]
```

**When to use:**
- **Flexible APIs**: Clients need different field sets
- **Mobile apps**: Reduce bandwidth for limited data
- **GraphQL-like**: Similar to GraphQL field selection
- **Performance**: Only fetch/serialize needed fields

---

#### Pattern 6: Multiple Serializers per Model

```python
class PostListSerializer(serializers.ModelSerializer):
    """
    Minimal serializer for list view
    Fast serialization, minimal data
    """
    author_name = serializers.CharField(source='author.username', read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'author_name', 'created_at']


class PostDetailSerializer(serializers.ModelSerializer):
    """
    Complete serializer for detail view
    All fields and relationships
    """
    author = UserSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    tags = TagSerializer(many=True, read_only=True)
    likes_count = serializers.IntegerField(read_only=True)
    is_liked = serializers.SerializerMethodField()
    
    class Meta:
        model = Post
        fields = [
            'id', 'title', 'content', 'author', 'comments', 
            'tags', 'likes_count', 'is_liked', 'created_at', 'updated_at'
        ]
    
    def get_is_liked(self, obj):
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.likes.filter(user=request.user).exists()
        return False


class PostCreateSerializer(serializers.ModelSerializer):
    """
    Writable serializer for creation
    Only accepts writable fields
    """
    tag_ids = serializers.ListField(
        child=serializers.IntegerField(),
        write_only=True,
        required=False
    )
    
    class Meta:
        model = Post
        fields = ['title', 'content', 'tag_ids']
    
    def create(self, validated_data):
        tag_ids = validated_data.pop('tag_ids', [])
        post = Post.objects.create(**validated_data)
        
        if tag_ids:
            post.tags.set(tag_ids)
        
        return post


class PostUpdateSerializer(serializers.ModelSerializer):
    """
    Update serializer with different validation rules
    """
    class Meta:
        model = Post
        fields = ['title', 'content', 'published']
    
    def validate_published(self, value):
        """Can't unpublish a post"""
        if self.instance and self.instance.published and not value:
            raise serializers.ValidationError(
                "Cannot unpublish a post"
            )
        return value


# Use in ViewSet
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    
    def get_serializer_class(self):
        """Return different serializer based on action"""
        if self.action == 'list':
            return PostListSerializer
        elif self.action == 'retrieve':
            return PostDetailSerializer
        elif self.action == 'create':
            return PostCreateSerializer
        elif self.action in ['update', 'partial_update']:
            return PostUpdateSerializer
        return PostSerializer  # Default
    
    def get_queryset(self):
        """Optimize queryset based on serializer needs"""
        queryset = super().get_queryset()
        
        if self.action == 'list':
            # Minimal fields for list serializer
            queryset = queryset.select_related('author').only(
                'id', 'title', 'author__username', 'created_at'
            )
        elif self.action == 'retrieve':
            # All fields for detail serializer
            queryset = queryset.select_related('author').prefetch_related(
                'comments', 'comments__author', 'tags', 'likes'
            ).annotate(likes_count=Count('likes'))
        
        return queryset
```

**When to use:**
- **Different contexts**: List vs detail vs create
- **Performance**: Optimize for each use case
- **Validation**: Different rules for create vs update
- **Enterprise**: Professional API design

---

#### Pattern 7: SerializerMethodField (Computed Fields)

```python
class PostSerializer(serializers.ModelSerializer):
    """
    Use SerializerMethodField for computed values
    """
    
    # Computed fields
    reading_time = serializers.SerializerMethodField()
    is_trending = serializers.SerializerMethodField()
    author_info = serializers.SerializerMethodField()
    relative_time = serializers.SerializerMethodField()
    
    class Meta:
        model = Post
        fields = [
            'id', 'title', 'content',
            'reading_time', 'is_trending', 'author_info', 'relative_time'
        ]
    
    def get_reading_time(self, obj):
        """
        Calculate reading time based on word count
        Executes for EACH object during serialization
        """
        word_count = len(obj.content.split())
        minutes = word_count // 200  # Average reading speed
        return f"{minutes} min read" if minutes > 0 else "< 1 min read"
    
    def get_is_trending(self, obj):
        """
        Check if post is trending
        ⚠️ WARNING: This can cause N+1 queries!
        """
        # Bad: Queries database for each post
        # recent_likes = obj.likes.filter(
        #     created_at__gte=timezone.now() - timedelta(days=1)
        # ).count()
        
        # Good: Use annotated value from queryset
        return getattr(obj, 'is_trending', False)
    
    def get_author_info(self, obj):
        """
        Return custom author information
        Uses select_related to avoid N+1
        """
        return {
            'id': obj.author.id,
            'username': obj.author.username,
            'is_verified': obj.author.is_verified,
            'posts_count': getattr(obj.author, 'posts_count', 0)
        }
    
    def get_relative_time(self, obj):
        """
        Human-readable relative time
        """
        from django.utils.timesince import timesince
        return f"{timesince(obj.created_at)} ago"


# Optimize in ViewSet
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return Post.objects.select_related('author').annotate(
            # Pre-compute trending status
            is_trending=Exists(
                Like.objects.filter(
                    post=OuterRef('pk'),
                    created_at__gte=timezone.now() - timedelta(days=1)
                ).values('pk')[:10]
            )
        )
```

**When to use:**
- **Computed values**: Not stored in database
- **Custom logic**: Complex calculations
- **Context-dependent**: Value depends on request user

**Performance tips:**
```python
# ❌ BAD: Causes queries
def get_comments_count(self, obj):
    return obj.comments.count()  # Query per object!

# ✅ GOOD: Use annotation
# In ViewSet: queryset.annotate(comments_count=Count('comments'))
comments_count = serializers.IntegerField(read_only=True)

# ❌ BAD: Heavy computation
def get_similarity_score(self, obj):
    # Complex ML calculation for each object
    return calculate_similarity(obj)

# ✅ GOOD: Pre-compute and store
# Add 'similarity_score' field to model, compute async
similarity_score = serializers.FloatField(read_only=True)
```

---

#### Pattern 8: Conditional Field Inclusion

```python
class ConditionalPostSerializer(serializers.ModelSerializer):
    """
    Include/exclude fields based on conditions
    """
    
    # Optional fields
    internal_notes = serializers.CharField(read_only=True)
    draft_content = serializers.CharField(read_only=True)
    author_email = serializers.EmailField(source='author.email', read_only=True)
    
    class Meta:
        model = Post
        fields = [
            'id', 'title', 'content',
            'internal_notes', 'draft_content', 'author_email'
        ]
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
        request = self.context.get('request')
        
        # Remove sensitive fields for non-staff users
        if request and not request.user.is_staff:
            self.fields.pop('internal_notes', None)
            self.fields.pop('draft_content', None)
            self.fields.pop('author_email', None)
        
        # Remove fields based on user permissions
        if request and not request.user.has_perm('posts.view_draft'):
            self.fields.pop('draft_content', None)


class PermissionAwareSerializer(serializers.ModelSerializer):
    """
    Advanced: Field-level permissions
    """
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
        request = self.context.get('request')
        if not request:
            return
        
        # Define field permissions
        field_permissions = {
            'salary': 'view_salary',
            'ssn': 'view_ssn',
            'internal_notes': 'view_internal',
        }
        
        # Remove fields user doesn't have permission for
        for field_name, permission in field_permissions.items():
            if not request.user.has_perm(f'posts.{permission}'):
                self.fields.pop(field_name, None)
```

**When to use:**
- **Role-based**: Different fields for different roles
- **Privacy**: Hide sensitive data
- **Permissions**: Field-level access control
- **Multi-tenant**: Different data per tenant

---

#### Pattern 9: Validation Patterns

```python
class AdvancedValidationSerializer(serializers.ModelSerializer):
    """
    Comprehensive validation patterns
    """
    
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    publish_date = serializers.DateTimeField(required=False)
    category = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all()
    )
    
    class Meta:
        model = Post
        fields = ['title', 'content', 'publish_date', 'category']
    
    # Pattern A: Field-level validation
    def validate_title(self, value):
        """
        Validate single field
        Runs during is_valid()
        """
        if len(value) < 10:
            raise serializers.ValidationError(
                "Title must be at least 10 characters"
            )
        
        # Check for profanity
        if contains_profanity(value):
            raise serializers.ValidationError(
                "Title contains inappropriate content"
            )
        
        # Check uniqueness with custom message
        if Post.objects.filter(title__iexact=value).exists():
            raise serializers.ValidationError(
                "A post with this title already exists"
            )
        
        return value.strip()  # Clean and return
    
    def validate_publish_date(self, value):
        """Validate publish date is in future"""
        if value and value < timezone.now():
            raise serializers.ValidationError(
                "Publish date must be in the future"
            )
        return value
    
    # Pattern B: Object-level validation
    def validate(self, data):
        """
        Validate multiple fields together
        Runs after all field validations pass
        """
        # Cross-field validation
        if data.get('publish_date') and not data.get('content'):
            raise serializers.ValidationError(
                "Cannot schedule empty post"
            )
        
        # Business rule validation
        if dataRemember: **Start simple, measure performance, optimize when needed**.

---

## Enterprise-Grade Django REST Framework Patterns

### Deep Dive: URL Patterns - 7 Different Approaches

#### Approach 1: Simple Router (Basic CRUD)

```python
# urls.py
from rest_framework.routers import SimpleRouter
from posts.views import PostViewSet

router = SimpleRouter()
router.register(r'posts', PostViewSet, basename='post')

urlpatterns = router.urls

# Generated URLs:
# /posts/          → list, create
# /posts/{pk}/     → retrieve, update, partial_update, destroy
```

**When to use:**
- Simple APIs with standard CRUD
- No need for API root view
- Microservices with single resource

**Question:** *Why no trailing slash option?*
**Answer:** SimpleRouter is opinionated. Use DefaultRouter for trailing slash control.

---

#### Approach 2: DefaultRouter (API Root + Options)

```python
# urls.py
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'posts', PostViewSet)
router.register(r'users', UserViewSet)
router.register(r'comments', CommentViewSet)

urlpatterns = router.urls

# Generated URLs:
# /                    → API root (lists all endpoints)
# /posts/              → PostViewSet
# /users/              → UserViewSet
# /comments/           → CommentViewSet
# Each with OPTIONS method support
```

**When to use:**
- Public APIs that need discoverability
- Multiple resources
- Browsable API with root endpoint
- Need OPTIONS support for CORS

**Question:** *What's the difference from SimpleRouter?*
**Answer:** DefaultRouter adds:
1. API root view at `/`
2. Trailing slash support
3. `.json` format suffix
4. Better for production APIs

---

#### Approach 3: Nested Routers (Parent-Child Resources)

```python
# Install: pip install drf-nested-routers

from rest_framework_nested import routers

router = routers.DefaultRouter()
router.register(r'posts', PostViewSet, basename='post')

# Nested router for post comments
posts_router = routers.NestedDefaultRouter(router, r'posts', lookup='post')
posts_router.register(r'comments', CommentViewSet, basename='post-comments')

urlpatterns = [
    path('api/', include(router.urls)),
    path('api/', include(posts_router.urls)),
]

# Generated URLs:
# /api/posts/                          → All posts
# /api/posts/{post_pk}/                → Single post
# /api/posts/{post_pk}/comments/       → Comments for post
# /api/posts/{post_pk}/comments/{pk}/  → Single comment
```

**ViewSet implementation:**

```python
class CommentViewSet(viewsets.ModelViewSet):
    serializer_class = CommentSerializer
    
    def get_queryset(self):
        # Access parent post_pk from URL
        post_pk = self.kwargs['post_pk']
        return Comment.objects.filter(post_id=post_pk)
    
    def perform_create(self, serializer):
        post_pk = self.kwargs['post_pk']
        serializer.save(post_id=post_pk, author=self.request.user)
```

**When to use:**
- Parent-child relationships (post → comments)
- RESTful resource nesting
- Clear hierarchical structure
- Better than: `/comments/?post_id=1`

**Question:** *Why not just use query parameters?*
**Answer:** 
- **Nested URLs**: Semantic, RESTful, clearer intent
- **Query params**: More flexible, better for filtering
- **Enterprise**: Use nested for core relationships, query params for filters

---

#### Approach 4: Manual URL Configuration (Full Control)

```python
# urls.py
from django.urls import path
from posts.views import (
    PostListCreateView,
    PostDetailView,
    PostPublishView,
    PostStatsView
)

urlpatterns = [
    path('posts/', PostListCreateView.as_view(), name='post-list'),
    path('posts/<int:pk>/', PostDetailView.as_view(), name='post-detail'),
    path('posts/<int:pk>/publish/', PostPublishView.as_view(), name='post-publish'),
    path('posts/stats/', PostStatsView.as_view(), name='post-stats'),
]

# Views
from rest_framework import generics

class PostListCreateView(generics.ListCreateAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

class PostDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

class PostPublishView(generics.GenericAPIView):
    queryset = Post.objects.all()
    
    def post(self, request, pk):
        post = self.get_object()
        post.published = True
        post.save()
        return Response({'status': 'published'})
```

**When to use:**
- Non-standard REST patterns
- Mixed ViewSet and APIView endpoints
- Custom URL patterns
- Fine-grained control needed

**Question:** *Isn't this more work than routers?*
**Answer:** Yes, but gives you:
- Custom URL patterns (`/posts/trending/`, `/posts/by-date/2024-01-01/`)
- Mix different view types
- No "magic" - explicit and clear
- Better for complex APIs

---

#### Approach 5: Versioned URLs (API Versioning)

```python
# urls.py - Approach A: URL Path Versioning
urlpatterns = [
    path('api/v1/', include('api.v1.urls')),
    path('api/v2/', include('api.v2.urls')),
]

# api/v1/urls.py
from rest_framework.routers import DefaultRouter
from .views import PostViewSetV1

router = DefaultRouter()
router.register(r'posts', PostViewSetV1)
urlpatterns = router.urls

# api/v2/urls.py
from rest_framework.routers import DefaultRouter
from .views import PostViewSetV2

router = DefaultRouter()
router.register(r'posts', PostViewSetV2)
urlpatterns = router.urls


# Approach B: Accept Header Versioning
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.AcceptHeaderVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
}

# Single URL, version in Accept header
# Request: Accept: application/json; version=v2

# urls.py
router = DefaultRouter()
router.register(r'posts', PostViewSet)

# views.py
class PostViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        if self.request.version == 'v2':
            return PostSerializerV2
        return PostSerializerV1


# Approach C: Query Parameter Versioning
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.QueryParameterVersioning',
}

# Request: GET /api/posts/?version=v2
```

**Comparison:**

| Method | Example | Pros | Cons | Enterprise Use |
|--------|---------|------|------|----------------|
| URL Path | `/api/v1/posts/` | Clear, cacheable | URL proliferation | ✅ Best for public APIs |
| Accept Header | `Accept: version=v2` | Clean URLs | Not browser-friendly | ✅ Internal services |
| Query Param | `?version=v2` | Simple | Breaks caching | ⚠️ Quick prototypes only |

**When to use:**
- Breaking changes in API
- Long-term API support
- Multiple client versions
- Enterprise requirement

---

#### Approach 6: Namespace URLs (Multi-tenant / Multi-app)

```python
# Main urls.py
urlpatterns = [
    path('api/blog/', include('blog.urls', namespace='blog')),
    path('api/shop/', include('shop.urls', namespace='shop')),
    path('api/forum/', include('forum.urls', namespace='forum')),
]

# blog/urls.py
app_name = 'blog'

router = DefaultRouter()
router.register(r'posts', PostViewSet, basename='post')

urlpatterns = [
    path('', include(router.urls)),
]

# Usage in code:
from django.urls import reverse

url = reverse('blog:post-detail', kwargs={'pk': 1})
# Result: /api/blog/posts/1/

# In serializers
class PostSerializer(serializers.HyperlinkedModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='blog:post-detail'
    )
```

**When to use:**
- Multi-tenant applications
- Separate Django apps
- Prevent URL name collisions
- Large monolithic projects

---

#### Approach 7: Dynamic URL Generation (Enterprise Pattern)

```python
# config/urls.py - Central configuration
from django.apps import apps
from rest_framework.routers import DefaultRouter

def generate_api_urls():
    """
    Automatically register all ViewSets from installed apps
    Enterprise pattern for large projects
    """
    router = DefaultRouter()
    
    for app_config in apps.get_app_configs():
        # Look for viewsets.py in each app
        try:
            viewsets_module = __import__(
                f'{app_config.name}.viewsets',
                fromlist=['']
            )
            
            # Auto-register viewsets with metadata
            for name in dir(viewsets_module):
                obj = getattr(viewsets_module, name)
                
                if (isinstance(obj, type) and 
                    issubclass(obj, viewsets.ModelViewSet) and
                    hasattr(obj, 'Meta')):
                    
                    prefix = obj.Meta.url_prefix
                    router.register(prefix, obj, basename=obj.Meta.basename)
        
        except (ImportError, AttributeError):
            continue
    
    return router.urls

urlpatterns = [
    path('api/', include(generate_api_urls())),
]

# posts/viewsets.py
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    
    class Meta:
        url_prefix = 'posts'
        basename = 'post'
```

**When to use:**
- Large enterprise applications (50+ apps)
- Plugin-based architecture
- Microservices monolith (modular monolith)
- Auto-discovery needed

**Question:** *Isn't auto-discovery "magic" and hard to debug?*
**Answer:** Yes! Trade-offs:
- **Pros**: DRY, scales to 100+ apps, plugin architecture
- **Cons**: Harder to debug, implicit behavior
- **Solution**: Use for stable enterprise apps with good documentation

---

### Deep Dive: View Patterns - 8 Different Approaches

#### Pattern 1: Function-Based Views (Simplest)

```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework import status

@api_view(['GET', 'POST'])
@permission_classes([IsAuthenticated])
def post_list(request):
    """
    Handle list and create operations
    
    Execution flow:
    1. @api_view wraps function
    2. Checks HTTP method against allowed methods
    3. Converts Django request → DRF Request
    4. Runs permission checks
    5. Executes function body
    6. Wraps return → DRF Response
    """
    
    if request.method == 'GET':
        posts = Post.objects.all()
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = PostSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


@api_view(['GET'])
def post_stats(request):
    """
    Custom endpoint: GET /api/posts/stats/
    """
    stats = {
        'total': Post.objects.count(),
        'published': Post.objects.filter(published=True).count(),
        'drafts': Post.objects.filter(published=False).count(),
        'by_author': list(
            Post.objects.values('author__username')
            .annotate(count=Count('id'))
        )
    }
    return Response(stats)
```

**When to use:**
- **Quick prototypes**: Fast to write
- **One-off endpoints**: Doesn't fit CRUD
- **Simple logic**: No complex state management
- **Utility endpoints**: Stats, exports, triggers

**Enterprise usage:**
```python
# Good for:
@api_view(['POST'])
def trigger_backup(request):
    backup_database.delay()  # Celery task
    return Response({'status': 'started'})

@api_view(['GET'])
def health_check(request):
    return Response({
        'status': 'healthy',
        'database': check_database(),
        'cache': check_cache(),
        'queue': check_queue()
    })
```

---

#### Pattern 2: APIView (Class-Based, Full Control)

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status, permissions

class PostList(APIView):
    """
    Class-based view with explicit method handlers
    
    Execution flow:
    1. URL dispatcher → PostList.as_view()
    2. as_view() returns view function
    3. Calls dispatch(request, *args, **kwargs)
    4. dispatch() → initial(request)
        - Runs authentication
        - Runs permissions
        - Runs throttling
    5. dispatch() → handler method (get/post/put/delete)
    6. Returns Response
    """
    
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    
    def get(self, request, format=None):
        """Handle GET /api/posts/"""
        posts = Post.objects.all()
        
        # Custom filtering logic
        status_filter = request.query_params.get('status')
        if status_filter:
            posts = posts.filter(status=status_filter)
        
        serializer = PostSerializer(posts, many=True, context={'request': request})
        return Response(serializer.data)
    
    def post(self, request, format=None):
        """Handle POST /api/posts/"""
        serializer = PostSerializer(data=request.data)
        
        if serializer.is_valid():
            # Custom business logic before save
            if request.user.post_count >= 10:
                return Response(
                    {'error': 'Post limit reached'},
                    status=status.HTTP_403_FORBIDDEN
                )
            
            serializer.save(author=request.user)
            
            # Custom business logic after save
            request.user.post_count += 1
            request.user.save()
            
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class PostDetail(APIView):
    """Handle single post operations"""
    
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    
    def get_object(self, pk):
        try:
            return Post.objects.get(pk=pk)
        except Post.DoesNotExist:
            raise Http404
    
    def get(self, request, pk, format=None):
        post = self.get_object(pk)
        serializer = PostSerializer(post, context={'request': request})
        return Response(serializer.data)
    
    def put(self, request, pk, format=None):
        post = self.get_object(pk)
        
        # Custom permission check
        if post.author != request.user:
            return Response(
                {'error': 'Not authorized'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        serializer = PostSerializer(post, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def delete(self, request, pk, format=None):
        post = self.get_object(pk)
        
        if post.author != request.user:
            return Response(
                {'error': 'Not authorized'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        post.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**When to use:**
- **Custom logic**: Business rules don't fit generic patterns
- **Non-CRUD**: Complex operations (multi-step workflows)
- **Mixed operations**: Different serializers per method
- **Fine control**: Need to customize every step

**Enterprise pattern:**

```python
class PaymentProcessView(APIView):
    """
    Complex multi-step operation
    Not CRUD, so APIView is perfect
    """
    permission_classes = [IsAuthenticated]
    
    def post(self, request):
        # Step 1: Validate payment data
        serializer = PaymentSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        
        with transaction.atomic():
            # Step 2: Create payment record
            payment = serializer.save(user=request.user)
            
            # Step 3: Call payment gateway
            gateway_response = stripe.charge(
                amount=payment.amount,
                source=request.data['token']
            )
            
            # Step 4: Update payment status
            payment.gateway_id = gateway_response['id']
            payment.status = 'completed'
            payment.save()
            
            # Step 5: Create order
            order = Order.objects.create(
                user=request.user,
                payment=payment,
                items=request.data['items']
            )
            
            # Step 6: Send confirmation
            send_confirmation_email.delay(order.id)
        
        return Response({
            'payment_id': payment.id,
            'order_id': order.id,
            'status': 'success'
        }, status=status.HTTP_201_CREATED)
```

---

#### Pattern 3: Generic Views (Pre-built Patterns)

```python
from rest_framework import generics

# Pattern A: Separate views for list/create and detail
class PostList(generics.ListCreateAPIView):
    """
    GET  /posts/  → list()
    POST /posts/  → create()
    
    Inheritance: ListCreateAPIView
      → ListModelMixin + CreateModelMixin + GenericAPIView
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    
    def perform_create(self, serializer):
        """Hook: Called after validation, before save"""
        serializer.save(author=self.request.user)


class PostDetail(generics.RetrieveUpdateDestroyAPIView):
    """
    GET    /posts/{pk}/  → retrieve()
    PUT    /posts/{pk}/  → update()
    PATCH  /posts/{pk}/  → partial_update()
    DELETE /posts/{pk}/  → destroy()
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]


# Pattern B: Granular control with specific mixins
from rest_framework import mixins, generics

class PostList(mixins.ListModelMixin,
               mixins.CreateModelMixin,
               generics.GenericAPIView):
    """
    Same as ListCreateAPIView but explicit about mixins
    Use when you need to understand/customize mixin behavior
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    
    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)
    
    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)


# Pattern C: Read-only view
class PostList(generics.ListAPIView):
    """Only GET /posts/ - no create"""
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    filter_backends = [filters.SearchFilter, filters.OrderingFilter]
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'title']
```

**Available Generic Views:**

```python
# Single action views
CreateAPIView          # POST only
ListAPIView            # GET list only
RetrieveAPIView        # GET detail only
DestroyAPIView         # DELETE only
UpdateAPIView          # PUT/PATCH only

# Combined views
ListCreateAPIView              # GET list + POST
RetrieveUpdateAPIView          # GET detail + PUT/PATCH
RetrieveDestroyAPIView         # GET detail + DELETE
RetrieveUpdateDestroyAPIView   # GET detail + PUT/PATCH + DELETE
```

**When to use:**
- **Standard CRUD**: Operations fit generic patterns
- **Less boilerplate**: Than APIView
- **More control**: Than ViewSet
- **No router needed**: Manual URL configuration

**Question:** *Generic views vs ViewSets - which is better?*
**Answer:**
- **Generic Views**: Better for non-standard REST patterns, manual URLs
- **ViewSets**: Better for standard CRUD, automatic URL routing
- **Enterprise**: Mix both based on needs

---

#### Pattern 4: ViewSets (Router-friendly)

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

class PostViewSet(viewsets.ModelViewSet):
    """
    Complete CRUD operations with router support
    
    Provides:
    - list()    → GET  /posts/
    - create()  → POST /posts/
    - retrieve() → GET  /posts/{pk}/
    - update()  → PUT  /posts/{pk}/
    - partial_update() → PATCH /posts/{pk}/
    - destroy() → DELETE /posts/{pk}/
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    filter_backends = [filters.SearchFilter, filters.OrderingFilter]
    search_fields = ['title', 'content']
    
    def get_queryset(self):
        """
        Customize queryset based on action
        Called for every request
        """
        queryset = super().get_queryset()
        
        if self.action == 'list':
            # Optimize list queries
            queryset = queryset.select_related('author').only(
                'id', 'title', 'author__username', 'created_at'
            )
        elif self.action == 'retrieve':
            # Full data for detail view
            queryset = queryset.select_related('author').prefetch_related('comments')
        
        return queryset
    
    def get_serializer_class(self):
        """Different serializers for different actions"""
        if self.action == 'list':
            return PostListSerializer
        elif self.action == 'retrieve':
            return PostDetailSerializer
        return PostSerializer
    
    def perform_create(self, serializer):
        """Hook before saving"""
        serializer.save(author=self.request.user)
    
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """
        Custom action: POST /posts/{pk}/publish/
        
        detail=True  → requires pk in URL
        detail=False → collection-level action
        """
        post = self.get_object()
        post.status = 'published'
        post.published_at = timezone.now()
        post.save()
        
        return Response({'status': 'published'})
    
    @action(detail=False, methods=['get'])
    def trending(self, request):
        """
        Custom action: GET /posts/trending/
        Collection-level action (no pk needed)
        """
        trending_posts = self.get_queryset().filter(
            created_at__gte=timezone.now() - timedelta(days=7)
        ).annotate(
            like_count=Count('likes')
        ).order_by('-like_count')[:10]
        
        serializer = self.get_serializer(trending_posts, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['post'], permission_classes=[permissions.IsAuthenticated])
    def like(self, request, pk=None):
        """
        POST /posts/{pk}/like/
        Custom permissions for specific action
        """
        post = self.get_object()
        Like.objects.get_or_create(user=request.user, post=post)
        return Response({'status': 'liked'})
```

**ViewSet Variations:**

```python
# 1. ModelViewSet - Full CRUD
class PostViewSet(viewsets.ModelViewSet):
    # Includes: list, create, retrieve, update, partial_update, destroy
    pass

# 2. ReadOnlyModelViewSet - Read-only
class PostViewSet(viewsets.ReadOnlyModelViewSet):
    # Includes: list, retrieve only
    pass

# 3. GenericViewSet - Custom combinations
class PostViewSet(viewsets.GenericViewSet):
    # No actions by default, add mixins as needed
    pass

# 4. GenericViewSet with specific mixins
class PostViewSet(mixins.ListModelMixin,
                  mixins.RetrieveModelMixin,
                  viewsets.GenericViewSet):
    # Only list and retrieve, no create/update/delete
    queryset = Post.objects.all()
    serializer_class = PostSerializer
```

**When to use:**
- **Standard REST**: CRUD operations
- **Router-based**: Automatic URL generation
- **Multiple actions**: Custom actions with @action
- **DRY code**: Minimal boilerplate

---

#### Pattern 5: Async Views (Django 4.1+, Real-time)

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from asgiref.sync import sync_to_async
import asyncio

class AsyncPostList(APIView):
    """
    Async view for concurrent operations
    Useful for: External API calls, I/O-bound operations
    """
    
    async def get(self, request):
        """
        Async GET handler
        Can await multiple operations concurrently
        """
        
        # Fetch data from multiple sources concurrently
        posts_task = sync_to_async(list)(
            Post.objects.select_related('author').all()
        )
        
        # External API call (async)
        async with aiohttp.ClientSession() as session:
            analytics_task = session.get('https://analytics.api/posts/stats')
            
            # Run concurrently
            posts, analytics_response = await asyncio.gather(
                posts_task,
                analytics_task
            )
        
        analytics_data = await analytics_response.json()
        
        # Serialize posts
        serializer = PostSerializer(posts, many=True)
        
        return Response({
            'posts': serializer.data,
            'analytics': analytics_data
        })
    
    async def post(self, request):
        """Async POST handler"""
        serializer = PostSerializer(data=request.data)
        
        # Sync operation wrapped in sync_to_async
        is_valid = await sync_to_async(serializer.is_valid)(raise_exception=True)
        
        # Save to database (sync operation)
        post = await sync_to_async(serializer.save)(author=request.user)
        
        # Trigger async tasks
        await asyncio.gather(
            send_notification_async(post.author, 'Post created'),
            update_search_index_async(post.id),
            notify_followers_async(post.id)
        )
        
        return Response(serializer.data, status=201)
```

**When to use:**
- **I/O-bound**: External API calls, file operations
- **Concurrent**: Multiple independent operations
- **Real-time**: WebSocket integration
- **High throughput**: Handle more concurrent requests

**Not for:**
- **CPU-bound**: Heavy computations (use Celery instead)
- **Simple CRUD**: Sync is simpler and sufficient

---

#### Pattern 6: Atomic Views (Transaction Safety)

```python
from django.db import transaction
from rest_framework import viewsets, status
from rest_framework.response import Response
from rest_framework.decorators import action

class OrderViewSet(viewsets.ModelViewSet):
    """
    Enterprise pattern: Ensure data consistency with transactions
    """
    queryset = Order.objects.all()
    serializer_class = OrderSerializer
    
    @transaction.atomic
    def create(self, request):
        """
        All database operations in atomic block
        If ANY operation fails, ALL rollback
        """
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        
        # Multiple related operations
        order = serializer.save(user=request.user)
        
        # Update inventory
        for item in request.data['items']:
            product = Product.objects.select_for_update().get(id=item['product_id'])
            
            if product.stock < item['quantity']:
                # This raise will rollback the entire transaction
                raise ValidationError(f"Insufficient stock for {product.name}")
            
            product.stock -= item['quantity']
            product.save()
        
        # Create payment record
        Payment.objects.create(
            order=order,
            amount=order.total,
            status='pending'
        )
        
        # Update user stats
        request.user.total_orders += 1
        request.user.save()
        
        # If we reach here, commit all changes
        return Response(serializer.data, status=status.HTTP_201_CREATED)
    
    @action(detail=True, methods=['post'])
    def cancel(self, request, pk=None):
        """Cancel order with automatic rollback"""
        with transaction.atomic():
            order = self.get_object()
            
            if order.status != 'pending':
                raise ValidationError("Cannot cancel processed order")
            
            # Restore inventory
            for item in order.items.all():
                product = Product.objects.select_for_update().get(id=item.product_id)
                product.stock += item.quantity
                product.save()
            
            # Refund payment
            if hasattr(order, 'payment'):
                order.payment.status = 'refunded'
                order.payment.save()
            
            # Update order status
            order.status = 'cancelled'
            order.save()
            
            # All changes committed together
            return Response({'status': 'cancelled'})
```

**Transaction decorators vs context managers:**

```python
# Method 1: Decorator (cleaner for entire method)
@transaction.atomic
def create(self, request):
    # Everything in this method is atomic
    pass

# Method 2: Context manager (fine-grained control)
def create(self, request):
    # Some non-atomic code here
    
    with transaction.atomic():
        # Only this block is atomic
        pass
    
    # More non-atomic code

# Method 3: Savepoints (nested transactions)
def complex_operation(self, request):
    with transaction.atomic():
        # Outer transaction
        
        try:
            with transaction.atomic():
                # Inner transaction (savepoint)
                risky_operation()
        except Exception:
            # Inner rolled back, outer continues
            pass
        
        safe_operation()
```

---

#### Pattern 7: Cached Views (Performance)

```python
from django.core.cache import cache
from# Django REST Framework: Complete Engineering Guide

## Table of Contents
1. [Request-Response Flow](#request-response-flow)
2. [URL Routing Engineering](#url-routing-engineering)
3. [Views: Execution Patterns](#views-execution-patterns)
4. [Serializers: Data Transformation](#serializers-data-transformation)
5. [Models: Database Layer](#models-database-layer)
6. [Renderers: Response Formatting](#renderers-response-formatting)
7. [First Principles & Design Patterns](#first-principles--design-patterns)
8. [Circle of Competence](#circle-of-competence)
9. [Efficient Code Organization](#efficient-code-organization)

---

## Request-Response Flow

### The Complete Journey: HTTP Request → Response

```
┌─────────────┐
│HTTP Request │  GET /api/posts/1/
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  WSGI Server    │  Gunicorn/uWSGI receives raw HTTP
│  (Gunicorn)     │  Parses headers, body, method
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│Django Middleware│  Process request (auth, CORS, etc.)
│     Stack       │  SecurityMiddleware, SessionMiddleware
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  URL Router     │  urls.py matches pattern
│  (URLconf)      │  Extracts path parameters: pk=1
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  DRF View       │  View.dispatch() → determine HTTP method
│  (ViewSet)      │  Run authentication & permissions
│                 │  Execute retrieve() method
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│   Queryset      │  ORM: Post.objects.get(pk=1)
│   Execution     │  SQL: SELECT * FROM posts WHERE id=1
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Model Layer    │  Post instance created
│  (ORM Object)   │  Python object with attributes
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Serializer     │  PostSerializer(post_instance)
│  (to_representation) Converts model → dict
│                 │  Handles nested relations
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│   Renderer      │  JSONRenderer converts dict → JSON
│  (JSONRenderer) │  Adds Content-Type header
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│DRF Response Obj │  Response(serializer.data)
│                 │  Sets status code, headers
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│Django Middleware│  Process response (reverse order)
│     Stack       │  Add headers, compress, etc.
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  HTTP Response  │  Status: 200 OK
│                 │  Body: {"id":1,"title":"..."}
└─────────────────┘
```

---

## URL Routing Engineering

### How URL Resolution Works

**File: `urls.py`**

```python
# myproject/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from posts.views import PostViewSet

# Router automatically generates URL patterns
router = DefaultRouter()
router.register(r'posts', PostViewSet, basename='post')

urlpatterns = [
    path('api/', include(router.urls)),
]
```

#### Line-by-Line Execution:

1. **`DefaultRouter()`** - Creates router instance
   - Router maintains registry: `{prefix: viewset}` mapping
   - Generates standard RESTful URL patterns

2. **`router.register(r'posts', PostViewSet, basename='post')`**
   - Registers ViewSet with 'posts' prefix
   - Creates URL patterns:
     ```
     /api/posts/          → list(), create()
     /api/posts/{pk}/     → retrieve(), update(), destroy()
     /api/posts/{pk}/custom_action/ → custom methods
     ```

3. **`include(router.urls)`** - Merges router URLs into main URLconf

### URL Resolution Process:

```python
# When request arrives: GET /api/posts/1/

# Step 1: Django matches 'api/' in main urls.py
# Step 2: Router matches 'posts/1/' pattern
# Step 3: Router extracts pk=1
# Step 4: Router determines method (GET) → calls retrieve()
# Step 5: View receives: retrieve(request, pk='1')
```

### Alternative URL Patterns (Manual):

```python
# Without Router - Manual URL definition
from posts.views import PostListCreateView, PostDetailView

urlpatterns = [
    path('api/posts/', PostListCreateView.as_view(), name='post-list'),
    path('api/posts/<int:pk>/', PostDetailView.as_view(), name='post-detail'),
]
```

**Differences:**
- **Router**: Auto-generates URLs, handles ViewSets, less boilerplate
- **Manual**: More control, works with APIView/GenericView, explicit

---

## Views: Execution Patterns

### Three Ways to Write Views

#### 1. Function-Based View (FBV)

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view(['GET', 'POST'])
def post_list(request):
    """
    Execution Flow:
    1. Decorator wraps function
    2. Converts Django request → DRF Request
    3. Executes function body
    4. Converts return value → DRF Response
    """
    if request.method == 'GET':
        posts = Post.objects.all()
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = PostSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=201)
        return Response(serializer.errors, status=400)
```

**Execution Path:**
```
@api_view decorator
  → wraps function
  → request arrives
  → check HTTP method against allowed methods
  → execute function
  → wrap return in Response object
```

**Pros:** Simple, explicit, easy to understand
**Cons:** More code duplication, less DRY

---

#### 2. Class-Based View (APIView)

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class PostList(APIView):
    """
    Execution Flow:
    1. dispatch() receives request
    2. Runs authentication_classes
    3. Runs permission_classes
    4. Calls appropriate method (get/post/put/delete)
    5. Returns Response
    """
    
    authentication_classes = [TokenAuthentication]
    permission_classes = [IsAuthenticated]
    
    def get(self, request, format=None):
        """
        When GET request arrives:
        1. dispatch() → self.initial(request) [auth/permissions]
        2. dispatch() → self.get(request)
        3. Returns Response object
        """
        posts = Post.objects.all()
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)
    
    def post(self, request, format=None):
        serializer = PostSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

**Execution Sequence:**
```python
# Internal flow for: POST /api/posts/

APIView.as_view()  # Returns view function
  ↓
dispatch(request, *args, **kwargs)
  ↓
self.initial(request)  # Authentication & Permissions
  ↓
  ├─ self.perform_authentication(request)
  │    → runs TokenAuthentication.authenticate()
  │    → sets request.user
  ↓
  ├─ self.check_permissions(request)
  │    → IsAuthenticated.has_permission()
  │    → raises PermissionDenied if False
  ↓
  └─ self.check_throttles(request)
       → rate limiting
  ↓
handler = getattr(self, request.method.lower())  # Gets self.post
  ↓
response = handler(request, *args, **kwargs)  # Executes self.post()
  ↓
self.response = self.finalize_response(request, response, *args, **kwargs)
  ↓
return response
```

**Pros:** Class structure, reusable, middleware hooks
**Cons:** More boilerplate than ViewSets

---

#### 3. Generic Views & ViewSets

```python
from rest_framework import viewsets, mixins
from rest_framework.decorators import action

class PostViewSet(viewsets.ModelViewSet):
    """
    ModelViewSet = GenericViewSet + all CRUD mixins
    
    Inheritance chain:
    ModelViewSet
      ↳ mixins.CreateModelMixin
      ↳ mixins.RetrieveModelMixin
      ↳ mixins.UpdateModelMixin
      ↳ mixins.DestroyModelMixin
      ↳ mixins.ListModelMixin
      ↳ GenericViewSet
          ↳ ViewSetMixin
          ↳ GenericAPIView
              ↳ APIView
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def get_queryset(self):
        """
        Called by get_object() and list()
        Execution: Every time queryset is accessed
        """
        queryset = super().get_queryset()
        
        # Optimize based on action
        if self.action == 'list':
            queryset = queryset.select_related('author')
        
        return queryset
    
    def perform_create(self, serializer):
        """
        Called by create() after validation
        Execution: POST request, after is_valid()
        """
        serializer.save(author=self.request.user)
    
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """
        Custom action: POST /api/posts/{pk}/publish/
        
        Execution flow:
        1. Router recognizes 'publish' action
        2. dispatch() routes to this method
        3. Method executes custom logic
        """
        post = self.get_object()  # Calls get_queryset(), checks permissions
        post.status = 'published'
        post.save()
        return Response({'status': 'published'})
```

**ViewSet Method Execution Order:**

```python
# For: GET /api/posts/1/

1. PostViewSet.as_view({'get': 'retrieve'})
   ↓ Router maps HTTP method → ViewSet method
   
2. dispatch(request, pk=1)
   ↓
   
3. initial(request)
   ↓ Run authentication & permissions
   
4. retrieve(request, pk=1)  # From RetrieveModelMixin
   ↓
   
5. get_object()
   ├─ get_queryset()  # Returns Post.objects.all()
   ├─ filter_queryset(queryset)  # Apply filters
   ├─ queryset.get(pk=1)  # Database query executes HERE
   └─ check_object_permissions(request, obj)
   ↓
   
6. get_serializer(instance)
   ↓
   
7. return Response(serializer.data)
```

**Comparison Table:**

| Feature | FBV | APIView | ViewSet |
|---------|-----|---------|---------|
| Lines of code | Most | Medium | Least |
| Flexibility | High | High | Medium |
| Reusability | Low | Medium | High |
| Router support | No | No | Yes |
| Auto URL generation | No | No | Yes |
| Learning curve | Easiest | Medium | Steepest |
| Best for | Simple endpoints | Custom logic | CRUD operations |

---

## Serializers: Data Transformation

### How Serialization Works

```python
from rest_framework import serializers

class PostSerializer(serializers.ModelSerializer):
    """
    Serializer has two main phases:
    1. Deserialization: JSON/Form data → Python objects → Model instances
    2. Serialization: Model instances → Python dict → JSON
    """
    
    author_name = serializers.CharField(source='author.username', read_only=True)
    comments_count = serializers.IntegerField(source='comments.count', read_only=True)
    url = serializers.HyperlinkedIdentityField(view_name='post-detail')
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author_name', 'comments_count', 'url']
        read_only_fields = ['id', 'created_at']
    
    def validate_title(self, value):
        """
        Field-level validation
        Execution: During is_valid() call, for 'title' field
        """
        if len(value) < 5:
            raise serializers.ValidationError("Title too short")
        return value
    
    def validate(self, data):
        """
        Object-level validation
        Execution: After all field validations pass
        """
        if 'title' in data and 'spam' in data['title'].lower():
            raise serializers.ValidationError("Spam detected")
        return data
    
    def create(self, validated_data):
        """
        Execution: When serializer.save() is called (no instance provided)
        """
        return Post.objects.create(**validated_data)
    
    def update(self, instance, validated_data):
        """
        Execution: When serializer.save() is called (instance provided)
        """
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.save()
        return instance
```

### Serialization Flow (Model → JSON):

```python
# Code: serializer = PostSerializer(post_instance)

Step 1: __init__(instance=post_instance)
  ↓ Stores instance
  ↓ Initializes fields
  
Step 2: Access .data property
  ↓
  
Step 3: to_representation(instance)  # Main conversion method
  ↓
  
  For each field in fields:
    ├─ field.get_attribute(instance)  # Gets model attribute
    │   Example: author_name → instance.author.username
    ↓
    ├─ field.to_representation(value)  # Converts to primitive
    │   CharField → str
    │   IntegerField → int
    │   DateTimeField → ISO format string
    ↓
    └─ result[field_name] = primitive_value
  
Step 4: Returns OrderedDict
  {
    'id': 1,
    'title': 'My Post',
    'author_name': 'john',
    'comments_count': 5
  }
```

### Deserialization Flow (JSON → Model):

```python
# Code: serializer = PostSerializer(data=request.data)
#       serializer.is_valid(raise_exception=True)
#       serializer.save()

Step 1: __init__(data={'title': 'New Post', 'content': '...'})
  ↓
  
Step 2: is_valid(raise_exception=True)
  ↓
  ├─ Run field validators
  │   ├─ validate_title('New Post')  # Field-level
  │   ├─ validate_content('...')
  │   └─ Store in validated_data
  ↓
  ├─ Run object validator
  │   └─ validate(validated_data)  # Object-level
  ↓
  └─ If any fail → raise ValidationError
  
Step 3: save()
  ↓
  ├─ Check if instance exists
  │   ├─ If NO → create(validated_data)
  │   └─ If YES → update(instance, validated_data)
  ↓
  └─ Return model instance
```

### Nested Serializers Execution:

```python
class CommentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Comment
        fields = ['id', 'text', 'author']

class PostSerializer(serializers.ModelSerializer):
    comments = CommentSerializer(many=True, read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'comments']
```

**Execution for nested serialization:**

```python
# post.comments.all() → [comment1, comment2, comment3]

PostSerializer(post).data
  ↓
  to_representation(post)
    ├─ title → post.title  # Simple field
    ↓
    ├─ comments → CommentSerializer(post.comments.all(), many=True)
    │   ↓
    │   to_representation([comment1, comment2, comment3])
    │     ├─ For comment1:
    │     │   └─ CommentSerializer(comment1).data
    │     ├─ For comment2:
    │     │   └─ CommentSerializer(comment2).data
    │     └─ For comment3:
    │         └─ CommentSerializer(comment3).data
    │   ↓
    │   Returns: [
    │     {'id': 1, 'text': '...', 'author': 'john'},
    │     {'id': 2, 'text': '...', 'author': 'jane'},
    │     {'id': 3, 'text': '...', 'author': 'bob'}
    │   ]
    └─ Result includes nested data
```

---

## Models: Database Layer

### ORM Query Execution

```python
from django.db import models

class Post(models.Model):
    """
    Model defines:
    1. Database table structure
    2. Field types and constraints
    3. Relationships
    4. Business logic methods
    """
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey('User', on_delete=models.CASCADE, related_name='posts')
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        db_table = 'posts'
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['created_at']),
            models.Index(fields=['author', 'created_at']),
        ]
    
    def __str__(self):
        return self.title
```

### Query Execution Path:

```python
# ViewSet: post = Post.objects.get(pk=1)

Step 1: Post.objects → Returns Manager instance
  ↓
  
Step 2: Manager.get(pk=1)
  ↓
  
Step 3: QuerySet created (lazy)
  QuerySet: SELECT * FROM posts WHERE id = 1
  ↓ NOT executed yet!
  
Step 4: QuerySet.get() forces evaluation
  ↓
  
Step 5: Django's SQL compiler
  ├─ Builds SQL query
  │   SELECT "posts"."id", "posts"."title", "posts"."content",
  │          "posts"."author_id", "posts"."created_at"
  │   FROM "posts"
  │   WHERE "posts"."id" = 1
  ↓
  
Step 6: Database executes query
  ↓
  
Step 7: Django creates Post instance
  post = Post(
    id=1,
    title='...',
    content='...',
    author_id=5,
    created_at=datetime(...)
  )
  ↓
  
Step 8: Returns Post object
```

### N+1 Query Problem:

```python
# BAD: Causes N+1 queries
posts = Post.objects.all()  # 1 query
for post in posts:
    print(post.author.username)  # N queries (one per post)

# Execution:
# Query 1: SELECT * FROM posts
# Query 2: SELECT * FROM users WHERE id = post1.author_id
# Query 3: SELECT * FROM users WHERE id = post2.author_id
# ... N more queries


# GOOD: Use select_related (for ForeignKey)
posts = Post.objects.select_related('author').all()  # 1 query with JOIN
for post in posts:
    print(post.author.username)  # No additional queries

# Execution:
# Query 1: SELECT * FROM posts INNER JOIN users ON posts.author_id = users.id


# GOOD: Use prefetch_related (for ManyToMany / Reverse FK)
posts = Post.objects.prefetch_related('comments').all()  # 2 queries

# Execution:
# Query 1: SELECT * FROM posts
# Query 2: SELECT * FROM comments WHERE post_id IN (1, 2, 3, ...)
# Django joins in Python
```

---

## Renderers: Response Formatting

### How Rendering Works

```python
from rest_framework.renderers import JSONRenderer, BrowsableAPIRenderer

class PostViewSet(viewsets.ModelViewSet):
    renderer_classes = [JSONRenderer, BrowsableAPIRenderer]
```

**Rendering Flow:**

```python
# View returns: Response({'id': 1, 'title': 'Post'})

Step 1: Response object created
  Response(
    data={'id': 1, 'title': 'Post'},
    status=200,
    headers={}
  )
  ↓
  
Step 2: finalize_response(request, response)
  ↓
  
Step 3: Perform content negotiation
  ├─ Check Accept header: Accept: application/json
  ├─ Match with available renderers
  └─ Select JSONRenderer
  ↓
  
Step 4: JSONRenderer.render(data, media_type, renderer_context)
  ├─ Convert OrderedDict to JSON string
  │   json.dumps({'id': 1, 'title': 'Post'})
  ↓
  └─ Returns: b'{"id":1,"title":"Post"}'
  
Step 5: Set response headers
  ├─ Content-Type: application/json
  └─ Content-Length: 28
  ↓
  
Step 6: Return HTTP response
  HTTP/1.1 200 OK
  Content-Type: application/json
  Content-Length: 28
  
  {"id":1,"title":"Post"}
```

### Custom Renderer:

```python
from rest_framework.renderers import BaseRenderer

class CSVRenderer(BaseRenderer):
    media_type = 'text/csv'
    format = 'csv'
    
    def render(self, data, media_type=None, renderer_context=None):
        """
        Execution: When Accept: text/csv header present
        """
        if not isinstance(data, list):
            data = [data]
        
        # Convert to CSV format
        csv_data = []
        if data:
            headers = ','.join(data[0].keys())
            csv_data.append(headers)
            
            for item in data:
                row = ','.join(str(v) for v in item.values())
                csv_data.append(row)
        
        return '\n'.join(csv_data).encode('utf-8')
```

---

## First Principles & Design Patterns

### First Principles Thinking

**Question: "Why does DRF need Serializers?"**

**First Principle Reasoning:**

1. **Foundation**: HTTP can only transmit text (bytes)
2. **Problem**: Python objects can't be sent over HTTP
3. **Core Need**: Conversion between Python ↔ Text
4. **Additional Needs**: 
   - Validation (data integrity)
   - Different representations (JSON, XML, CSV)
   - Security (field-level permissions)

**Result**: Serializer = Converter + Validator + Presenter

### Design Patterns in DRF

#### 1. **Mixin Pattern**

```python
# Instead of inheritance hell, compose behaviors

class ListModelMixin:
    def list(self, request, *args, **kwargs):
        queryset = self.filter_queryset(self.get_queryset())
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)

class CreateModelMixin:
    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)
        return Response(serializer.data, status=201)

# Compose what you need
class PostViewSet(ListModelMixin, CreateModelMixin, GenericViewSet):
    pass  # Gets list() and create() methods
```

**Why Mixins?**
- **Principle**: Composition over inheritance
- **Benefit**: Pick exactly what you need
- **Flexibility**: Easy to override specific methods

#### 2. **Adapter Pattern**

```python
# DRF adapts Django's request/response to REST semantics

# Django HttpRequest → DRF Request (adapter)
class Request:
    def __init__(self, request):
        self._request = request  # Wraps Django request
        self._data = None
        
    @property
    def data(self):
        """Adapter: Unifies POST, PUT, PATCH data"""
        if self._data is None:
            self._data = self._parse()
        return self._data
```

#### 3. **Strategy Pattern**

```python
# Authentication strategies

class TokenAuthentication(BaseAuthentication):
    def authenticate(self, request):
        token = request.META.get('HTTP_AUTHORIZATION')
        # Strategy: Token-based auth
        return (user, token)

class SessionAuthentication(BaseAuthentication):
    def authenticate(self, request):
        # Strategy: Session-based auth
        return (request.user, None)

# View chooses strategy
class PostViewSet(viewsets.ModelViewSet):
    authentication_classes = [TokenAuthentication]  # Strategy selection
```

---

## Circle of Competence

### Level 1: Beginner (Focus Area)
**Master these first:**

```python
# 1. Simple serializers
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content']

# 2. Basic ViewSets
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

# 3. Simple URL routing
router.register(r'posts', PostViewSet)
```

**Why start here?**
- Covers 80% of use cases
- Builds foundation for advanced concepts
- Quick wins, immediate productivity

### Level 2: Intermediate (Expand Gradually)

```python
# Custom actions, filtering, permissions
class PostViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticatedOrReadOnly]
    filter_backends = [SearchFilter]
    search_fields = ['title']
    
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        # Custom business logic
        pass
```

### Level 3: Advanced (Master Later)

```python
# Custom renderers, complex nested serializers, optimization
class OptimizedPostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return Post.objects.select_related('author').prefetch_related(
            Prefetch('comments', queryset=Comment.objects.select_related('user'))
        )
```

**Progression Strategy:**
1. Master Level 1 thoroughly (2-3 months)
2. Identify real problems that need Level 2 solutions
3. Only explore Level 3 when you have specific performance needs

---

## Efficient Code Organization

### Project Structure

```
myproject/
├── myproject/
│   ├── settings.py
│   ├── urls.py              # Main URL router
│   └── wsgi.py
│
├── apps/
│   ├── posts/
│   │   ├── models.py        # Database models
│   │   ├── serializers.py   # Serializer classes
│   │   ├── views.py         # ViewSets and Views
│   │   ├── urls.py          # App-specific URLs
│   │   ├── permissions.py   # Custom permissions
│   │   ├── filters.py       # Custom filters
│   │   └── tests.py         # Unit tests
│   │
│   └── users/
│       └── ...
│
├── core/
│   ├── permissions.py       # Shared permissions
│   ├── pagination.py        # Custom pagination
│   └── renderers.py         # Custom renderers
│
└── manage.py
```

### When to Use Each Pattern

#### Use **Function-Based Views** when:
```python
# Simple, one-off endpoints
# No reuse needed
# Quick prototypes

@api_view(['POST'])
def trigger_export(request):
    # One-time utility endpoint
    export_data.delay()
    return Response({'status': 'started'})
```

#### Use **APIView** when:
```python
# Custom logic not fitting CRUD
# Multiple related operations
# Need class structure but not CRUD

class StatsView(APIView):
    def get(self, request):
        # Complex aggregation logic
        stats = calculate_statistics()
        return Response(stats)
```

#### Use **GenericAPIView + Mixins** when:
```python
# Need some CRUD, but not all
# Custom combinations

class PostListCreate(mixins.ListModelMixin,
                     mixins.CreateModelMixin,
                     generics.GenericAPIView):
    # Only list and create, no detail views
    pass
```

#### Use **ViewSets** when:
```python
# Full CRUD operations
# Standard RESTful resource
# Want router auto-generation

class PostViewSet(viewsets.ModelViewSet):
    # Complete CRUD + custom actions
    queryset = Post.objects.all()
    serializer_class = PostSerializer
```

### Efficiency Rules

1. **Don't Optimize Prematurely**
   ```python
   # Start simple
   queryset = Post.objects.all()
   
   # Optimize when you measure a problem
   queryset = Post.objects.select_related('author').prefetch_related('tags')
   ```

2. **Measure First, Then Optimize**
   ```python
   from django.db import connection
   from django.test.utils import override_settings
   
   @override_settings(DEBUG=True)
   def test_query_count():
       Post.objects.all()[0].author.username
       print(len(connection.queries))  # Measure queries
   ```

3. **Use the Simplest Solution**
   ```python
   # Too complex for simple task
   class PostViewSet(viewsets.ModelViewSet):
       # ... 50 lines of code
   
   # Better for simple need
   @api_view(['GET'])
   def get_posts(request):
       posts = Post.objects.all()
       return Response(PostSerializer(posts, many=True).data)
   ```

---

## Deep Dive: Request-Response Lifecycle

### Complete Execution Trace

Let's trace a real request: `POST /api/posts/` with authentication.

```python
# Request payload:
POST /api/posts/ HTTP/1.1
Host: api.example.com
Authorization: Token abc123xyz
Content-Type: application/json

{
  "title": "My New Post",
  "content": "This is the content"
}
```

#### Execution Timeline:

```python
# ==================== PHASE 1: SERVER LAYER ====================
# Time: 0ms

# 1. Gunicorn/uWSGI receives raw socket data
worker.handle_request(socket_data)

# 2. Parse HTTP protocol
http_parser.parse(socket_data)
  → method: POST
  → path: /api/posts/
  → headers: {'Authorization': 'Token abc123xyz', 'Content-Type': 'application/json'}
  → body: '{"title":"My New Post","content":"This is the content"}'


# ==================== PHASE 2: DJANGO MIDDLEWARE ====================
# Time: 2ms

# 3. Django creates HttpRequest object
request = HttpRequest()
request.method = 'POST'
request.path = '/api/posts/'
request.META['HTTP_AUTHORIZATION'] = 'Token abc123xyz'
request.body = b'{"title":"My New Post","content":"This is the content"}'

# 4. Middleware stack (top to bottom)
SecurityMiddleware(request)          # HTTPS redirect, security headers
SessionMiddleware(request)           # Load session data
CommonMiddleware(request)            # URL handling, ETags
CsrfViewMiddleware(request)          # CSRF protection (skipped for API)
AuthenticationMiddleware(request)    # Sets request.user (basic Django auth)
MessageMiddleware(request)           # Flash messages
XFrameOptionsMiddleware(request)     # Clickjacking protection


# ==================== PHASE 3: URL RESOLUTION ====================
# Time: 4ms

# 5. URLconf resolution
urlpatterns = [
    path('api/', include(router.urls)),  # Match found!
]

# 6. Router pattern matching
router.urls = [
    path('posts/', PostViewSet.as_view({'get': 'list', 'post': 'create'})),
    path('posts/<pk>/', PostViewSet.as_view({'get': 'retrieve', ...})),
]

# Match: 'posts/' → PostViewSet
# Method: POST → action='create'

# 7. View instantiation
view_func = PostViewSet.as_view({'post': 'create'})
# This returns a callable that wraps the ViewSet


# ==================== PHASE 4: DRF REQUEST WRAPPER ====================
# Time: 6ms

# 8. DRF wraps Django request
from rest_framework.request import Request

drf_request = Request(
    request=django_request,
    parsers=[JSONParser(), FormParser(), MultiPartParser()],
    authenticators=[TokenAuthentication()],
    negotiator=DefaultContentNegotiation(),
    parser_context={'view': view, 'args': (), 'kwargs': {}}
)

# 9. Parse request data (lazy - happens on first access to request.data)
drf_request.data  # Triggers parsing
  ↓
parser = JSONParser()
parsed_data = parser.parse(
    stream=request.body,
    media_type='application/json',
    parser_context={}
)
# Result: {'title': 'My New Post', 'content': 'This is the content'}


# ==================== PHASE 5: VIEW DISPATCH ====================
# Time: 8ms

# 10. ViewSet.dispatch() called
PostViewSet.dispatch(request=drf_request)

# 11. Initialize the view
self.action = 'create'
self.format_kwarg = None
self.args = ()
self.kwargs = {}

# 12. initial() - Pre-processing
self.initial(request)
  ↓
  # 12a. Authentication
  self.perform_authentication(request)
    ↓
    authenticator = TokenAuthentication()
    user, token = authenticator.authenticate(request)
      ↓
      # Extract token from header
      auth_header = request.META.get('HTTP_AUTHORIZATION')  # 'Token abc123xyz'
      token_key = auth_header.split(' ')[1]  # 'abc123xyz'
      
      # Database query
      token_obj = Token.objects.select_related('user').get(key='abc123xyz')
      # SQL: SELECT * FROM authtoken_token 
      #      INNER JOIN auth_user ON authtoken_token.user_id = auth_user.id
      #      WHERE authtoken_token.key = 'abc123xyz'
      
      user = token_obj.user
      request.user = user  # Set authenticated user
      request.auth = token_obj
  
  # 12b. Permission check
  self.check_permissions(request)
    ↓
    for permission in self.permission_classes:
        perm = IsAuthenticatedOrReadOnly()
        if not perm.has_permission(request, self):
            raise PermissionDenied()
    # User is authenticated → Permission granted
  
  # 12c. Throttling
  self.check_throttles(request)
    ↓
    throttles = [UserRateThrottle()]
    for throttle in throttles:
        if not throttle.allow_request(request, self):
            raise Throttled()
    # Check rate: user made 50 requests today, limit is 1000 → Allowed


# ==================== PHASE 6: ACTION EXECUTION ====================
# Time: 12ms

# 13. Dispatch to action handler
handler = getattr(self, 'create')  # Get create() method
response = handler(request)

# 14. CreateModelMixin.create() executes
def create(self, request, *args, **kwargs):
    # 14a. Get serializer
    serializer = self.get_serializer(data=request.data)
    # PostSerializer(data={'title': 'My New Post', 'content': '...'})
    
    # 14b. Validation
    serializer.is_valid(raise_exception=True)


# ==================== PHASE 7: SERIALIZER VALIDATION ====================
# Time: 15ms

# 15. Serializer.is_valid() execution
PostSerializer.is_valid(raise_exception=True)
  ↓
  # 15a. Field validation
  self._validated_data = {}
  self._errors = {}
  
  for field_name, field in self.fields.items():
      # Validate 'title' field
      if field_name == 'title':
          primitive_value = self.initial_data.get('title')  # 'My New Post'
          
          # Run field validators
          field.run_validation(primitive_value)
            ↓
            # Check required
            if field.required and primitive_value is None:
                raise ValidationError('This field is required')
            
            # Check max_length
            if len(primitive_value) > field.max_length:
                raise ValidationError('Ensure this field has no more than...')
            
            # Custom field validator
            value = self.validate_title(primitive_value)
              ↓
              if len(value) < 5:
                  raise ValidationError("Title too short")
              return value
          
          self._validated_data['title'] = value
      
      # Same for 'content' field
      if field_name == 'content':
          # ... validation process ...
          self._validated_data['content'] = validated_value
  
  # 15b. Object-level validation
  self._validated_data = self.validate(self._validated_data)
    ↓
    if 'spam' in data.get('title', '').lower():
        raise ValidationError('Spam detected')
    return data
  
  # 15c. Check for errors
  if self._errors:
      raise ValidationError(self._errors)
  
  # Validation passed!
  self.validated_data = {'title': 'My New Post', 'content': 'This is the content'}


# ==================== PHASE 8: DATABASE OPERATION ====================
# Time: 18ms

# 16. Save to database
serializer.save()
  ↓
  # No instance provided → calls create()
  self.perform_create(serializer)
    ↓
    serializer.save(author=request.user)
      ↓
      validated_data['author'] = request.user
      instance = self.create(validated_data)
        ↓
        # ModelSerializer.create()
        post = Post.objects.create(**validated_data)
          ↓
          # Django ORM
          post = Post(
              title='My New Post',
              content='This is the content',
              author=request.user
          )
          post.save()
            ↓
            # SQL generation
            cursor.execute(
                """
                INSERT INTO posts (title, content, author_id, created_at)
                VALUES (%s, %s, %s, %s)
                RETURNING id
                """,
                ['My New Post', 'This is the content', 5, datetime.now()]
            )
            
            # Database executes INSERT
            # Returns: id=42
            
            post.id = 42
      
      return post  # Post instance with id=42


# ==================== PHASE 9: SERIALIZATION (Model → Dict) ====================
# Time: 22ms

# 17. Return Response with serialized data
return Response(serializer.data, status=status.HTTP_201_CREATED)
  ↓
  # Access serializer.data triggers to_representation()
  serializer.data
    ↓
    self.to_representation(instance=post)
      ↓
      ret = OrderedDict()
      
      for field_name, field in self.fields.items():
          # Get attribute from model
          attribute = field.get_attribute(instance)
          
          if field_name == 'id':
              attribute = instance.id  # 42
              ret['id'] = field.to_representation(attribute)  # 42
          
          if field_name == 'title':
              attribute = instance.title  # 'My New Post'
              ret['title'] = field.to_representation(attribute)  # 'My New Post'
          
          if field_name == 'content':
              attribute = instance.content
              ret['content'] = field.to_representation(attribute)
          
          if field_name == 'created_at':
              attribute = instance.created_at  # datetime object
              ret['created_at'] = field.to_representation(attribute)
              # Converts to ISO format: '2025-10-06T10:30:00Z'
          
          if field_name == 'author':
              attribute = instance.author  # User object
              # If author is serialized as nested object
              ret['author'] = field.to_representation(attribute)
              # Returns: {'id': 5, 'username': 'john'}
      
      return ret
      # Result:
      # OrderedDict([
      #     ('id', 42),
      #     ('title', 'My New Post'),
      #     ('content', 'This is the content'),
      #     ('created_at', '2025-10-06T10:30:00Z'),
      #     ('author', {'id': 5, 'username': 'john'})
      # ])


# ==================== PHASE 10: RESPONSE CREATION ====================
# Time: 24ms

# 18. Create Response object
response = Response(
    data=OrderedDict([...]),
    status=201,
    headers={'Location': '/api/posts/42/'}
)


# ==================== PHASE 11: CONTENT NEGOTIATION ====================
# Time: 25ms

# 19. finalize_response()
self.finalize_response(request, response)
  ↓
  # 19a. Content negotiation
  negotiator = DefaultContentNegotiation()
  renderer, media_type = negotiator.select_renderer(
      request=request,
      renderers=self.renderer_classes,
      format_suffix=None
  )
  
  # Check Accept header
  accept_header = request.META.get('HTTP_ACCEPT')  # 'application/json'
  
  # Match with available renderers
  for renderer_class in [JSONRenderer, BrowsableAPIRenderer]:
      if renderer_class.media_type == 'application/json':
          selected_renderer = JSONRenderer()
          break
  
  response.accepted_renderer = selected_renderer
  response.accepted_media_type = 'application/json'


# ==================== PHASE 12: RENDERING ====================
# Time: 26ms

# 20. Render response
rendered_content = response.rendered_content
  ↓
  renderer = JSONRenderer()
  content = renderer.render(
      data=response.data,
      accepted_media_type='application/json',
      renderer_context={
          'view': self,
          'request': request,
          'response': response
      }
  )
  ↓
  # JSONRenderer.render()
  import json
  
  # Handle special types (datetime, Decimal, etc.)
  json_string = json.dumps(
      response.data,
      cls=DjangoJSONEncoder,
      ensure_ascii=False
  )
  
  encoded = json_string.encode('utf-8')
  # Returns: b'{"id":42,"title":"My New Post",...}'


# ==================== PHASE 13: HTTP RESPONSE BUILDING ====================
# Time: 28ms

# 21. Build HTTP response
http_response = HttpResponse(
    content=rendered_content,
    status=201,
    content_type='application/json'
)

# 22. Add headers
http_response['Content-Type'] = 'application/json'
http_response['Content-Length'] = len(rendered_content)
http_response['Location'] = '/api/posts/42/'
http_response['Vary'] = 'Accept'
http_response['Allow'] = 'GET, POST, HEAD, OPTIONS'


# ==================== PHASE 14: MIDDLEWARE RESPONSE ====================
# Time: 29ms

# 23. Response middleware (bottom to top)
XFrameOptionsMiddleware(request, response)   # Add X-Frame-Options
MessageMiddleware(request, response)         # Process messages
AuthenticationMiddleware(request, response)  # Nothing to do
CsrfViewMiddleware(request, response)        # Nothing to do
CommonMiddleware(request, response)          # Add ETag
SessionMiddleware(request, response)         # Save session
SecurityMiddleware(request, response)        # Add security headers


# ==================== PHASE 15: SEND RESPONSE ====================
# Time: 30ms

# 24. Serialize HTTP response
http_response_bytes = b"""HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 156
Location: /api/posts/42/
Vary: Accept
Allow: GET, POST, HEAD, OPTIONS
X-Frame-Options: DENY
X-Content-Type-Options: nosniff

{"id":42,"title":"My New Post","content":"This is the content","created_at":"2025-10-06T10:30:00Z","author":{"id":5,"username":"john"}}"""

# 25. Send to client
socket.send(http_response_bytes)

# ==================== COMPLETE ====================
# Total Time: 30ms
```

### Performance Breakdown

```
Phase                          Time    Percentage
─────────────────────────────────────────────────
Server/Middleware              4ms     13%
URL Resolution                 2ms     7%
DRF Request Wrapping           2ms     7%
Authentication                 4ms     13%
Validation                     3ms     10%
Database INSERT                6ms     20%
Serialization                  4ms     13%
Rendering                      2ms     7%
Response Building              3ms     10%
─────────────────────────────────────────────────
Total                          30ms    100%
```

**Optimization Opportunities:**

1. **Database (20%)** - Biggest impact
   - Use database indexes
   - Optimize queries
   - Connection pooling

2. **Authentication (13%)** - Cache tokens
   - Redis for token storage
   - Reduce DB lookups

3. **Serialization (13%)** - Only serialize needed fields
   - Use `fields` parameter
   - Avoid over-fetching

---

## Advanced Patterns & Trade-offs

### Pattern 1: Flat vs Nested Serializers

#### Flat Structure (Recommended for APIs)
```python
class PostSerializer(serializers.ModelSerializer):
    author_id = serializers.IntegerField()
    author_name = serializers.CharField(source='author.username', read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author_id', 'author_name']

# Output:
{
    "id": 1,
    "title": "My Post",
    "content": "...",
    "author_id": 5,
    "author_name": "john"
}
```

**Pros:**
- Easier to consume for clients
- Clear data structure
- Better for caching
- Simpler validation

**Cons:**
- Data duplication if used in multiple places
- May require multiple requests for related data

#### Nested Structure
```python
class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email']

class PostSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author']

# Output:
{
    "id": 1,
    "title": "My Post",
    "content": "...",
    "author": {
        "id": 5,
        "username": "john",
        "email": "john@example.com"
    }
}
```

**Pros:**
- Complete data in one request
- Intuitive structure
- Good for read-heavy operations

**Cons:**
- Complex write operations
- Harder to cache
- Can lead to over-fetching
- Performance issues with deep nesting

**Decision Framework:**
```python
# Use FLAT when:
# - Building public APIs
# - Need high performance
# - Clients vary in needs
# - Write operations are common

# Use NESTED when:
# - Internal microservices
# - Read-only or mostly read
# - Tight coupling with clients
# - UI needs complete data
```

### Pattern 2: Multiple Serializers per ViewSet

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    
    def get_serializer_class(self):
        """
        Dynamic serializer selection based on action
        
        Execution: Called before serializer instantiation
        """
        if self.action == 'list':
            # List: Minimal data, optimized for performance
            return PostListSerializer
        elif self.action == 'retrieve':
            # Detail: Complete data with relationships
            return PostDetailSerializer
        elif self.action == 'create':
            # Create: Writable fields only
            return PostCreateSerializer
        elif self.action == 'update':
            # Update: Different validation rules
            return PostUpdateSerializer
        return PostSerializer  # Default
    
    def get_queryset(self):
        """
        Dynamic queryset optimization based on action
        
        Execution: Every time queryset is accessed
        """
        queryset = super().get_queryset()
        
        if self.action == 'list':
            # List: Only needed fields, no relations
            queryset = queryset.only('id', 'title', 'created_at')
        elif self.action == 'retrieve':
            # Detail: Optimize with select_related
            queryset = queryset.select_related('author').prefetch_related('comments')
        
        return queryset


# Serializers
class PostListSerializer(serializers.ModelSerializer):
    """Minimal data for list view"""
    class Meta:
        model = Post
        fields = ['id', 'title', 'created_at']

class PostDetailSerializer(serializers.ModelSerializer):
    """Complete data for detail view"""
    author = UserSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'comments', 'created_at']

class PostCreateSerializer(serializers.ModelSerializer):
    """Writable fields for creation"""
    class Meta:
        model = Post
        fields = ['title', 'content']
    
    def create(self, validated_data):
        # Auto-assign author from request
        validated_data['author'] = self.context['request'].user
        return super().create(validated_data)
```

**Why this pattern?**
- **Principle**: Single Responsibility - each serializer does one thing well
- **Benefit**: Optimized for specific use case
- **Trade-off**: More code, but better performance

### Pattern 3: Serializer Context Usage

```python
class PostSerializer(serializers.ModelSerializer):
    is_owner = serializers.SerializerMethodField()
    can_edit = serializers.SerializerMethodField()
    url = serializers.SerializerMethodField()
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'is_owner', 'can_edit', 'url']
    
    def get_is_owner(self, obj):
        """
        Execution: For each object during serialization
        Context available: request, view, format
        """
        request = self.context.get('request')
        if not request or not request.user.is_authenticated:
            return False
        return obj.author == request.user
    
    def get_can_edit(self, obj):
        request = self.context.get('request')
        view = self.context.get('view')
        
        # Check permission without making extra queries
        permission = IsAuthorOrReadOnly()
        return permission.has_object_permission(request, view, obj)
    
    def get_url(self, obj):
        request = self.context.get('request')
        # Build absolute URL
        return request.build_absolute_uri(f'/api/posts/{obj.id}/')


# In view, context is automatically passed
class PostViewSet(viewsets.ModelViewSet):
    def list(self, request):
        posts = self.get_queryset()
        serializer = self.get_serializer(posts, many=True)
        # Context automatically includes: {'request': request, 'view': self, 'format': None}
        return Response(serializer.data)
```

**Context Flow:**
```
ViewSet.get_serializer()
  ↓
  context = {
      'request': self.request,
      'format': self.format_kwarg,
      'view': self
  }
  ↓
  Serializer(__init__, context=context)
  ↓
  Available in SerializerMethodField via self.context
```

---

## Practical Efficiency Guidelines

### Rule 1: Start Simple, Optimize When Measured

```python
# ❌ PREMATURE OPTIMIZATION (Don't do this initially)
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        # Too complex from the start
        return Post.objects.select_related(
            'author', 'author__profile', 'category'
        ).prefetch_related(
            Prefetch('comments', queryset=Comment.objects.select_related('user')),
            Prefetch('tags', queryset=Tag.objects.only('name')),
            'likes', 'likes__user'
        ).annotate(
            comments_count=Count('comments'),
            likes_count=Count('likes')
        ).only('id', 'title', 'author__username')

# ✅ START SIMPLE
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

# ✅ MEASURE PERFORMANCE
# Use Django Debug Toolbar or django-silk
# Identify N+1 queries

# ✅ OPTIMIZE SPECIFIC PROBLEMS
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        queryset = Post.objects.all()
        
        # Optimize only what you measured
        if self.action == 'list':
            queryset = queryset.select_related('author')
        
        return queryset
```

### Rule 2: Choose the Right Tool

```python
# Use Case: Simple CRUD with standard behavior
# ✅ BEST: ModelViewSet (minimal code)
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer


# Use Case: Need only list and detail, no create/update/delete
# ✅ BEST: ReadOnlyModelViewSet
class PostViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer


# Use Case: Custom business logic, not standard CRUD
# ✅ BEST: APIView
class ExportPostsView(APIView):
    def post(self, request):
        # Custom logic that doesn't fit CRUD pattern
        format = request.data.get('format')
        posts = Post.objects.all()
        
        if format == 'csv':
            return self.export_csv(posts)
        elif format == 'pdf':
            return self.export_pdf(posts)
        
        return Response({'error': 'Invalid format'}, status=400)


# Use Case: Single utility endpoint
# ✅ BEST: Function-based view
@api_view(['POST'])
@permission_classes([IsAdminUser])
def rebuild_search_index(request):
    from tasks import rebuild_index
    rebuild_index.delay()
    return Response({'status': 'started'})


# Use Case: Complex multi-step operation
# ✅ BEST: Custom ViewSet method
class PostViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    def publish_and_notify(self, request, pk=None):
        """
        Complex operation: publish post and send notifications
        Not a standard CRUD operation
        """
        post = self.get_object()
        post.status = 'published'
        post.published_at = timezone.now()
        post.save()
        
        # Send notifications
        notify_followers.delay(post.id)
        
        return Response({
            'status': 'published',
            'published_at': post.published_at
        })
```

### Rule 3: Understand Query Execution

```python
# Database queries execute at EVALUATION time, not creation time

# ❌ INEFFICIENT: Queries execute in loop
def get_posts_with_author_names():
    posts = Post.objects.all()  # Query NOT executed yet (lazy)
    result = []
    
    for post in posts:  # Query executes here
        author_name = post.author.username  # N+1: New query for each post!
        result.append({
            'title': post.title,
            'author': author_name
        })
    
    return result

# Execution:
# Query 1: SELECT * FROM posts
# Query 2: SELECT * FROM users WHERE id = 1
# Query 3: SELECT * FROM users WHERE id = 2
# ... 100 queries for 100 posts!


# ✅ EFFICIENT: Single query with join
def get_posts_with_author_names():
    posts = Post.objects.select_related('author').all()
    result = []
    
    for post in posts:  # Still only ONE query
        author_name = post.author.username  # No additional query
        result.append({
            'title': post.title,
            'author': author_name
        })
    
    return result

# Execution:
# Query 1: SELECT * FROM posts 
#          INNER JOIN users ON posts.author_id = users.id
# Total: 1 query


# Understanding query execution timing:
queryset = Post.objects.all()  # NO query yet
queryset = queryset.filter(published=True)  # Still NO query
queryset = queryset.select_related('author')  # STILL no query

# Query executes when you:
list(queryset)  # Convert to list
queryset[0]  # Access by index
for post in queryset:  # Iterate
    pass
len(queryset)  # Get count
bool(queryset)  # Check if exists
```

### Rule 4: DRY (Don't Repeat Yourself) vs Explicit

```python
# Balance between DRY and explicit code

# ❌ TOO DRY (Hard to understand)
class BaseSerializer(serializers.ModelSerializer):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # Magic happens here, but unclear what
        self.process_dynamic_fields()
        self.apply_conditional_validation()
        self.setup_context_based_fields()
    
    # ... 200 lines of abstraction

class PostSerializer(BaseSerializer):
    # No idea what this does without reading BaseSerializer
    class Meta:
        model = Post
        fields = '__all__'


# ❌ NOT DRY ENOUGH (Repetitive)
class PostCreateSerializer(serializers.ModelSerializer):
    def validate_title(self, value):
        if len(value) < 5:
            raise serializers.ValidationError("Title too short")
        if 'spam' in value.lower():
            raise serializers.ValidationError("Spam detected")
        return value
    
    class Meta:
        model = Post
        fields = ['title', 'content']

class PostUpdateSerializer(serializers.ModelSerializer):
    def validate_title(self, value):
        # Same validation repeated!
        if len(value) < 5:
            raise serializers.ValidationError("Title too short")
        if 'spam' in value.lower():
            raise serializers.ValidationError("Spam detected")
        return value
    
    class Meta:
        model = Post
        fields = ['title', 'content', 'status']


# ✅ BALANCED: DRY but still explicit
class TitleValidationMixin:
    """Clear, reusable validation"""
    def validate_title(self, value):
        if len(value) < 5:
            raise serializers.ValidationError("Title too short")
        if 'spam' in value.lower():
            raise serializers.ValidationError("Spam detected")
        return value

class PostCreateSerializer(TitleValidationMixin, serializers.ModelSerializer):
    # Clear what it does, validation is separate
    class Meta:
        model = Post
        fields = ['title', 'content']

class PostUpdateSerializer(TitleValidationMixin, serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['title', 'content', 'status']
```

---

## Common Mistakes & Solutions

### Mistake 1: Over-fetching Data

```python
# ❌ WRONG: Fetching unnecessary data
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = '__all__'  # Returns ALL fields including large content

# API returns:
{
    "id": 1,
    "title": "Post",
    "content": "... 10,000 characters ...",  # Wasted bandwidth for list view
    "raw_html": "...",  # Not needed
    "internal_notes": "...",  # Should never be exposed
    "password_hash": "..."  # SECURITY ISSUE!
}


# ✅ CORRECT: Explicit fields for different contexts
class PostListSerializer(serializers.ModelSerializer):
    """Minimal data for list view"""
    class Meta:
        model = Post
        fields = ['id', 'title', 'excerpt', 'created_at']  # Only what's needed

class PostDetailSerializer(serializers.ModelSerializer):
    """Full data for detail view"""
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'created_at', 'updated_at']
        # Explicit exclusion is safer than fields = '__all__'


# ✅ OPTIMAL: Use .only() for database efficiency
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        queryset = Post.objects.all()
        
        if self.action == 'list':
            # Only fetch fields we'll serialize
            queryset = queryset.only('id', 'title', 'excerpt', 'created_at')
        
        return queryset
```

### Mistake 2: Not Handling Errors Properly

```python
# ❌ WRONG: Generic error handling
class PostViewSet(viewsets.ModelViewSet):
    def create(self, request):
        try:
            serializer = PostSerializer(data=request.data)
            serializer.save()  # No validation!
            return Response(serializer.data)
        except Exception as e:
            return Response({'error': 'Something went wrong'}, status=500)
        # Lost all error details!


# ✅ CORRECT: Let DRF handle validation errors
class PostViewSet(viewsets.ModelViewSet):
    def create(self, request):
        serializer = PostSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)  # Raises ValidationError with details
        serializer.save()
        return Response(serializer.data, status=status.HTTP_201_CREATED)


# ✅ BETTER: Handle specific exceptions explicitly
class PostViewSet(viewsets.ModelViewSet):
    def create(self, request):
        serializer = PostSerializer(data=request.data)
        
        try:
            serializer.is_valid(raise_exception=True)
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        
        except serializers.ValidationError:
            # Let DRF handle validation errors (returns 400 with details)
            raise
        
        except IntegrityError as e:
            # Database constraint violation
            return Response(
                {'error': 'Duplicate entry or constraint violation'},
                status=status.HTTP_409_CONFLICT
            )
        
        except PermissionDenied:
            # Re-raise permission errors (returns 403)
            raise
        
        except Exception as e:
            # Log unexpected errors
            logger.exception("Unexpected error in post creation")
            return Response(
                {'error': 'An unexpected error occurred'},
                status=status.HTTP_500_INTERNAL_SERVER_ERROR
            )
```

### Mistake 3: Not Using Transactions

```python
# ❌ WRONG: Multiple save operations without transaction
class PostViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    def publish_with_notification(self, request, pk=None):
        post = self.get_object()
        
        # If this succeeds...
        post.status = 'published'
        post.save()
        
        # But this fails...
        notification = Notification.objects.create(
            user=post.author,
            message=f"Your post '{post.title}' is published"
        )  # CRASH! But post is already saved!
        
        # Database is now inconsistent: post published but no notification
        return Response({'status': 'published'})


# ✅ CORRECT: Use atomic transactions
from django.db import transaction

class PostViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    @transaction.atomic  # Decorator approach
    def publish_with_notification(self, request, pk=None):
        post = self.get_object()
        
        post.status = 'published'
        post.save()
        
        Notification.objects.create(
            user=post.author,
            message=f"Your post '{post.title}' is published"
        )
        
        # If ANY exception occurs, BOTH operations rollback
        return Response({'status': 'published'})
    
    # Alternative: Context manager approach
    def create(self, request):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        
        with transaction.atomic():
            # All database operations in this block are atomic
            post = serializer.save()
            
            # Additional operations
            post.author.post_count += 1
            post.author.save()
            
            Tag.objects.bulk_create([
                Tag(post=post, name=tag)
                for tag in request.data.get('tags', [])
            ])
        
        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

### Mistake 4: Ignoring Caching Opportunities

```python
# ❌ INEFFICIENT: Hitting database on every request
class PostViewSet(viewsets.ModelViewSet):
    def retrieve(self, request, pk=None):
        post = Post.objects.get(pk=pk)
        serializer = self.get_serializer(post)
        return Response(serializer.data)
    # Every request hits database, even for unchanged data


# ✅ GOOD: Basic caching
from django.core.cache import cache

class PostViewSet(viewsets.ModelViewSet):
    def retrieve(self, request, pk=None):
        cache_key = f'post:{pk}'
        cached_data = cache.get(cache_key)
        
        if cached_data:
            return Response(cached_data)
        
        post = Post.objects.get(pk=pk)
        serializer = self.get_serializer(post)
        
        # Cache for 15 minutes
        cache.set(cache_key, serializer.data, 60 * 15)
        
        return Response(serializer.data)
    
    def update(self, request, pk=None):
        # Invalidate cache on update
        cache.delete(f'post:{pk}')
        return super().update(request, pk=pk)


# ✅ BETTER: Versioned caching (cache invalidation)
class PostViewSet(viewsets.ModelViewSet):
    def retrieve(self, request, pk=None):
        post = Post.objects.only('updated_at').get(pk=pk)
        
        # Cache key includes version (updated_at timestamp)
        cache_key = f'post:{pk}:{post.updated_at.timestamp()}'
        cached_data = cache.get(cache_key)
        
        if cached_data:
            return Response(cached_data)
        
        # Re-fetch with all fields
        post = Post.objects.get(pk=pk)
        serializer = self.get_serializer(post)
        
        # Cache indefinitely (new version will have different key)
        cache.set(cache_key, serializer.data, None)
        
        return Response(serializer.data)


# ✅ BEST: Use Django's cache decorators
from django.views.decorators.cache import cache_page
from django.utils.decorators import method_decorator

class PostViewSet(viewsets.ModelViewSet):
    @method_decorator(cache_page(60 * 15))  # Cache for 15 minutes
    def list(self, request):
        return super().list(request)
```

### Mistake 5: Inefficient Pagination

```python
# ❌ WRONG: No pagination (returns ALL records)
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()  # Could be 100,000 records!
    serializer_class = PostSerializer
    # Client receives HUGE response, slow serialization


# ⚠️ PROBLEMATIC: PageNumberPagination for large datasets
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    pagination_class = PageNumberPagination
    
    # Problem: Offset pagination is slow for deep pages
    # SELECT * FROM posts LIMIT 20 OFFSET 10000
    # Database must scan 10,020 rows to return 20


# ✅ GOOD: CursorPagination for large datasets
from rest_framework.pagination import CursorPagination

class PostCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    pagination_class = PostCursorPagination
    
    # Uses WHERE created_at < '...' instead of OFFSET
    # Much faster for large datasets


# ✅ OPTIMAL: Custom pagination with optimization
class OptimizedPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100
    
    def get_paginated_response(self, data):
        return Response({
            'count': self.page.paginator.count,
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'results': data,
            # Additional metadata
            'page_size': self.page_size,
            'current_page': self.page.number,
            'total_pages': self.page.paginator.num_pages
        })
```

---

## Testing Strategies

### Unit Testing Serializers

```python
from django.test import TestCase
from rest_framework.test import APITestCase

class PostSerializerTest(TestCase):
    def setUp(self):
        self.user = User.objects.create(username='testuser')
        self.post = Post.objects.create(
            title='Test Post',
            content='Test content',
            author=self.user
        )
    
    def test_serialization(self):
        """Test model → dict conversion"""
        serializer = PostSerializer(self.post)
        data = serializer.data
        
        self.assertEqual(data['title'], 'Test Post')
        self.assertEqual(data['content'], 'Test content')
        self.assertEqual(data['author_id'], self.user.id)
    
    def test_deserialization_valid(self):
        """Test dict → model with valid data"""
        data = {
            'title': 'New Post',
            'content': 'New content',
            'author_id': self.user.id
        }
        
        serializer = PostSerializer(data=data)
        self.assertTrue(serializer.is_valid())
        post = serializer.save()
        
        self.assertEqual(post.title, 'New Post')
        self.assertIsInstance(post, Post)
    
    def test_validation_title_too_short(self):
        """Test validation error"""
        data = {
            'title': 'Hi',  # Too short
            'content': 'Content',
            'author_id': self.user.id
        }
        
        serializer = PostSerializer(data=data)
        self.assertFalse(serializer.is_valid())
        self.assertIn('title', serializer.errors)
    
    def test_read_only_fields(self):
        """Test that read-only fields can't be written"""
        data = {
            'id': 999,  # Should be ignored
            'title': 'Post',
            'content': 'Content',
            'author_id': self.user.id
        }
        
        serializer = PostSerializer(data=data)
        serializer.is_valid(raise_exception=True)
        post = serializer.save()
        
        self.assertNotEqual(post.id, 999)  # id was auto-generated
```

### Integration Testing ViewSets

```python
from rest_framework.test import APITestCase, APIClient
from rest_framework import status

class PostViewSetTest(APITestCase):
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        # Authenticate for protected endpoints
        self.client.force_authenticate(user=self.user)
        
        self.post = Post.objects.create(
            title='Test Post',
            content='Test content',
            author=self.user
        )
    
    def test_list_posts(self):
        """Test GET /api/posts/"""
        response = self.client.get('/api/posts/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 1)
        self.assertEqual(response.data['results'][0]['title'], 'Test Post')
    
    def test_create_post(self):
        """Test POST /api/posts/"""
        data = {
            'title': 'New Post',
            'content': 'New content'
        }
        
        response = self.client.post('/api/posts/', data)
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Post.objects.count(), 2)
        self.assertEqual(response.data['title'], 'New Post')
    
    def test_create_post_unauthenticated(self):
        """Test POST /api/posts/ without authentication"""
        self.client.force_authenticate(user=None)  # Remove authentication
        
        data = {'title': 'New Post', 'content': 'Content'}
        response = self.client.post('/api/posts/', data)
        
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
    
    def test_update_own_post(self):
        """Test PUT /api/posts/{id}/ - owner can update"""
        data = {'title': 'Updated Title', 'content': 'Updated content'}
        
        response = self.client.put(f'/api/posts/{self.post.id}/', data)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.post.refresh_from_db()
        self.assertEqual(self.post.title, 'Updated Title')
    
    def test_update_others_post(self):
        """Test PUT /api/posts/{id}/ - non-owner cannot update"""
        other_user = User.objects.create_user(
            username='otheruser',
            password='pass123'
        )
        self.client.force_authenticate(user=other_user)
        
        data = {'title': 'Hacked Title', 'content': 'Hacked'}
        response = self.client.put(f'/api/posts/{self.post.id}/', data)
        
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
    
    def test_delete_post(self):
        """Test DELETE /api/posts/{id}/"""
        response = self.client.delete(f'/api/posts/{self.post.id}/')
        
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertEqual(Post.objects.count(), 0)
    
    def test_custom_action(self):
        """Test POST /api/posts/{id}/publish/"""
        response = self.client.post(f'/api/posts/{self.post.id}/publish/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.post.refresh_from_db()
        self.assertTrue(self.post.published)


class PostQueryOptimizationTest(APITestCase):
    """Test query count to ensure optimization"""
    
    def setUp(self):
        self.user = User.objects.create_user(username='user', password='pass')
        self.client.force_authenticate(user=self.user)
        
        # Create test data
        for i in range(10):
            Post.objects.create(
                title=f'Post {i}',
                content=f'Content {i}',
                author=self.user
            )
    
    def test_list_query_count(self):
        """Ensure list view doesn't cause N+1 queries"""
        with self.assertNumQueries(2):  # 1 for posts, 1 for count
            response = self.client.get('/api/posts/')
            self.assertEqual(len(response.data['results']), 10)
```

---

## Performance Optimization Checklist

### Database Level

```python
# ✅ 1. Use select_related for ForeignKey
Post.objects.select_related('author', 'category')

# ✅ 2. Use prefetch_related for ManyToMany and reverse ForeignKey
Post.objects.prefetch_related('tags', 'comments')

# ✅ 3. Use only() to fetch specific fields
Post.objects.only('id', 'title', 'created_at')

# ✅ 4. Use defer() to exclude heavy fields
Post.objects.defer('content', 'raw_html')

# ✅ 5. Use values() for dictionaries (faster than model instances)
Post.objects.values('id', 'title')  # Returns dict, not model

# ✅ 6. Use database indexes
class Post(models.Model):
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['author', 'created_at']),
            models.Index(fields=['-created_at']),  # For ordering
        ]

# ✅ 7. Use bulk operations
Post.objects.bulk_create([
    Post(title=f'Post {i}', content=f'Content {i}')
    for i in range(1000)
])  # Single query instead of 1000

# ✅ 8. Use update() instead of save() for bulk updates
Post.objects.filter(author=user).update(status='published')
# Instead of:
# for post in Post.objects.filter(author=user):
#     post.status = 'published'
#     post.save()
```

### Serializer Level

```python
# ✅ 1. Use SerializerMethodField sparingly (expensive)
class PostSerializer(serializers.ModelSerializer):
    # ❌ BAD: Causes N+1
    comments_count = serializers.SerializerMethodField()
    
    def get_comments_count(self, obj):
        return obj.comments.count()  # Query for EACH post
    
    # ✅ GOOD: Annotate in queryset
    # In ViewSet:
    # queryset = Post.objects.annotate(comments_count=Count('comments'))
    comments_count = serializers.IntegerField(read_only=True)

# ✅ 2. Use source for simple field access
class PostSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source='author.username', read_only=True)
    # Faster than SerializerMethodField

# ✅ 3. Avoid nested writes unless necessary
# Nested reads are fine, but nested writes add complexity

# ✅ 4. Use ListSerializer for bulk operations
class PostListSerializer(serializers.ListSerializer):
    def create(self, validated_data):
        posts = [Post(**item) for item in validated_data]
        return Post.objects.bulk_create(posts)

class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = '__all__'
        list_serializer_class = PostListSerializer
```

### View Level

```python
# ✅ 1. Implement caching
@method_decorator(cache_page(60 * 15))
def list(self, request):
    return super().list(request)

# ✅ 2. Use pagination
pagination_class = CursorPagination

# ✅ 3. Implement filtering to reduce dataset
filter_backends = [DjangoFilterBackend, SearchFilter]
filterset_fields = ['status', 'author']

# ✅ 4. Use throttling to prevent abuse
throttle_classes = [UserRateThrottle]

# ✅ 5. Optimize queryset per action
def get_queryset(self):
    queryset = super().get_queryset()
    
    if self.action == 'list':
        queryset = queryset.select_related('author').only('id', 'title')
    elif self.action == 'retrieve':
        queryset = queryset.select_related('author').prefetch_related('comments')
    
    return queryset
```

---

## Summary: Decision Tree

```
Need an API endpoint?
│
├─ Standard CRUD operations?
│  │
│  ├─ YES → Use ModelViewSet
│  │      queryset = Model.objects.all()
│  │      serializer_class = ModelSerializer
│  │
│  └─ NO → Custom logic?
│     │
│     ├─ Multiple operations → Use APIView
│     └─ Single operation → Use @api_view
│
├─ Need authentication?
│  │
│  ├─ Token-based → authentication_classes = [TokenAuthentication]
│  ├─ Session-based → authentication_classes = [SessionAuthentication]
│  └─ JWT → Use djangorestframework-simplejwt
│
├─ Need permissions?
│  │
│  ├─ Simple → permission_classes = [IsAuthenticated]
│  └─ Complex → Create custom permission class
│
├─ Performance concerns?
│  │
│  ├─ Database → Use select_related, prefetch_related, only()
│  ├─ Serialization → Different serializers per action
│  └─ Response time → Implement caching
│
└─ Testing?
   │
   ├─ Serializers → TestCase with serializer.is_valid()
   └─ Views → APITestCase with self.client.get/post()
```

---

## Final Principles

### 1. **First Principles: Build from Foundation**
- HTTP can only send text
- Need conversion: Objects ↔ Text
- Need validation: Ensure data integrity
- Need abstraction: Reduce boilerplate
- **Result**: Serializers + Views + Routers

### 2. **Circle of Competence: Master Gradually**
- **Level 1**: ModelViewSet + ModelSerializer (80% of use cases)
- **Level 2**: Custom actions, permissions, filtering
- **Level 3**: Performance optimization, custom renderers

### 3. **Efficiency: Measure, Then Optimize**
- Start simple
- Measure performance (Django Debug Toolbar)
- Optimize bottlenecks
- Don't premature optimize

### 4. **Code Quality: Balance DRY and Clarity**
- Reuse common patterns (mixins, base classes)
- Keep code explicit and readable
- Document non-obvious decisions
- Write tests for critical paths

---

## Quick Reference

```python
# Minimal working API
from rest_framework import viewsets, serializers
from rest_framework.routers import DefaultRouter

class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content']

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

router = DefaultRouter()
router.register(r'posts', PostViewSet)
urlpatterns = router.urls
```

**That's it!** This gives you a fully functional REST API with:
- GET /posts/ (list)
- POST /posts/ (create)
- GET /posts/{id}/ (retrieve)
- PUT /posts/{id}/ (update)
- PATCH /posts/{id}/ (partial update)
- DELETE /posts/{id}/ (delete)

Build from here based on your specific needs.

---

## Additional Resources

- **Official Docs**: https://www.django-rest-framework.org/
- **Source Code**: https://github.com/encode/django-rest-framework
- **Django ORM**: https://docs.djangoproject.com/en/stable/topics/db/queries/
- **Testing**: https://www.django-rest-framework.org/api-guide/testing/

Remember: **Start simple, measure performance, optimize when needed**., value):
        raise ValidationError("Only alphanumeric characters allowed")

class PostSerializer(serializers.ModelSerializer):
    title = serializers.CharField(
        max_length=200,
        validators=[
            UniqueValidator(queryset=Post.objects.all()),
            validate_no_special_chars
        ]
    )
    
    class Meta:
        model = Post
        fields = ['title', 'content']
        validators = [
            # Unique together validation
            serializers.UniqueTogetherValidator(
                queryset=Post.objects.all(),
                fields=['title', 'author']
            )
        ]


# Pattern E: Async validation (for external services)
class ExternalValidationSerializer(serializers.ModelSerializer):
    email = serializers.EmailField()
    
    def validate_email(self, value):
        """Validate email with external service"""
        # Check if email exists via external API
        # In real async view, this would be await
        is_valid = verify_email_external(value)
        
        if not is_valid:
            raise serializers.ValidationError(
                "Email address could not be verified"
            )
        
        return value
```

---

#### Pattern 10: Polymorphic Serializers (Different Types)

```python
from rest_framework import serializers

# Base models
class Content(models.Model):
    title = models.CharField(max_length=200)
    created_at = models.DateTimeField(auto_now_add=True)
    content_type = models.CharField(max_length=50)  # 'article', 'video', 'podcast'
    
    class Meta:
        abstract = False

class Article(Content):
    text = models.TextField()
    read_time = models.IntegerField()

class Video(Content):
    video_url = models.URLField()
    duration = models.IntegerField()

class Podcast(Content):
    audio_url = models.URLField()
    episode_number = models.IntegerField()


# Serializers
class ArticleSerializer(serializers.ModelSerializer):
    type = serializers.SerializerMethodField()
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'type', 'text', 'read_time', 'created_at']
    
    def get_type(self, obj):
        return 'article'

class VideoSerializer(serializers.ModelSerializer):
    type = serializers.SerializerMethodField()
    
    class Meta:
        model = Video
        fields = ['id', 'title', 'type', 'video_url', 'duration', 'created_at']
    
    def get_type(self, obj):
        return 'video'

class PodcastSerializer(serializers.ModelSerializer):
    type = serializers.SerializerMethodField()
    
    class Meta:
        model = Podcast
        fields = ['id', 'title', 'type', 'audio_url', 'episode_number', 'created_at']
    
    def get_type(self, obj):
        return 'podcast'


# Polymorphic Serializer
class ContentSerializer(serializers.Serializer):
    """
    Automatically selects correct serializer based on type
    """
    
    def to_representation(self, instance):
        """Serialize based on instance type"""
        if isinstance(instance, Article):
            serializer = ArticleSerializer(instance, context=self.context)
        elif isinstance(instance, Video):
            serializer = VideoSerializer(instance, context=self.context)
        elif isinstance(instance, Podcast):
            serializer = PodcastSerializer(instance, context=self.context)
        else:
            raise ValueError(f"Unsupported content type: {type(instance)}")
        
        return serializer.data
    
    def to_internal_value(self, data):
        """Deserialize based on type field"""
        content_type = data.get('type')
        
        if content_type == 'article':
            serializer = ArticleSerializer(data=data, context=self.context)
        elif content_type == 'video':
            serializer = VideoSerializer(data=data, context=self.context)
        elif content_type == 'podcast':
            serializer = PodcastSerializer(data=data, context=self.context)
        else:
            raise serializers.ValidationError({
                'type': f"Invalid content type: {content_type}"
            })
        
        serializer.is_valid(raise_exception=True)
        return serializer.validated_data


# ViewSet
class ContentViewSet(viewsets.ModelViewSet):
    """
    Returns mixed content types
    """
    serializer_class = ContentSerializer
    
    def get_queryset(self):
        # Return union of all content types
        from itertools import chain
        
        articles = Article.objects.all()
        videos = Video.objects.all()
        podcasts = Podcast.objects.all()
        
        # Combine and sort
        content = sorted(
            chain(articles, videos, podcasts),
            key=lambda x: x.created_at,
            reverse=True
        )
        
        return content

# Response:
[
    {
        "id": 1,
        "title": "My Article",
        "type": "article",
        "text": "...",
        "read_time": 5
    },
    {
        "id": 2,
        "title": "My Video",
        "type": "video",
        "video_url": "https://...",
        "duration": 120
    },
    {
        "id": 3,
        "title": "My Podcast",
        "type": "podcast",
        "audio_url": "https://...",
        "episode_number": 42
    }
]
```

**When to use:**
- **Mixed content**: Feed with different types
- **Inheritance**: Django model inheritance
- **Flexible APIs**: Support multiple resource types
- **Activity feeds**: Posts, comments, likes, etc.

---

### Deep Dive: Model Patterns - 8 Enterprise Approaches

#### Pattern 1: Abstract Base Models

```python
from django.db import models
from django.contrib.auth.models import User

class TimeStampedModel(models.Model):
    """
    Abstract base class for timestamp tracking
    Inherit from this for auto timestamps
    """
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True  # Won't create database table
        ordering = ['-created_at']
        get_latest_by = 'created_at'


class SoftDeleteModel(models.Model):
    """
    Soft delete pattern - don't actually delete records
    """
    is_deleted = models.BooleanField(default=False, db_index=True)
    deleted_at = models.DateTimeField(null=True, blank=True)
    deleted_by = models.ForeignKey(
        User,
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name='+'
    )
    
    class Meta:
        abstract = True
    
    def delete(self, using=None, keep_parents=False):
        """Override delete to soft delete"""
        self.is_deleted = True
        self.deleted_at = timezone.now()
        self.save()
    
    def hard_delete(self):
        """Actually delete from database"""
        super().delete()


class AuditModel(models.Model):
    """
    Track who created/modified records
    """
    created_by = models.ForeignKey(
        User,
        on_delete=models.SET_NULL,
        null=True,
        related_name='%(class)s_created'
    )
    updated_by = models.ForeignKey(
        User,
        on_delete=models.SET_NULL,
        null=True,
        related_name='%(class)s_updated'
    )
    
    class Meta:
        abstract = True


# Use all base models
class Post(TimeStampedModel, SoftDeleteModel, AuditModel):
    """
    Inherits: created_at, updated_at, is_deleted, created_by, updated_by
    """
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')
    
    def __str__(self):
        return self.title


# Custom manager for soft deletes
class SoftDeleteManager(models.Manager):
    """Manager that excludes soft-deleted records"""
    
    def get_queryset(self):
        return super().get_queryset().filter(is_deleted=False)
    
    def deleted(self):
        """Get only deleted records"""
        return super().get_queryset().filter(is_deleted=True)
    
    def with_deleted(self):
        """Get all records including deleted"""
        return super().get_queryset()


class Post(TimeStampedModel, SoftDeleteModel):
    # ... fields ...
    
    objects = SoftDeleteManager()  # Default manager excludes deleted
    all_objects = models.Manager()  # Access all including deleted
    
    class Meta:
        base_manager_name = 'all_objects'  # For admin


# Usage:
Post.objects.all()  # Only non-deleted
Post.all_objects.all()  # Including deleted
Post.objects.deleted()  # Only deleted
```

**When to use:**
- **DRY**: Reuse common fields across models
- **Consistency**: Standard timestamps, soft deletes
- **Audit trails**: Track who did what
- **Enterprise**: Required for compliance

---

#### Pattern 2: Custom Managers & QuerySets

```python
class PostQuerySet(models.QuerySet):
    """
    Custom queryset with business logic methods
    Chainable and reusable
    """
    
    def published(self):
        """Get only published posts"""
        return self.filter(status='published', published_at__lte=timezone.now())
    
    def drafts(self):
        """Get only drafts"""
        return self.filter(status='draft')
    
    def by_author(self, author):
        """Get posts by specific author"""
        return self.filter(author=author)
    
    def trending(self, days=7):
        """Get trending posts"""
        since = timezone.now() - timedelta(days=days)
        return self.published().annotate(
            recent_likes=Count(
                'likes',
                filter=Q(likes__created_at__gte=since)
            )
        ).filter(recent_likes__gte=10).order_by('-recent_likes')
    
    def with_stats(self):
        """Annotate with statistics"""
        return self.annotate(
            likes_count=Count('likes'),
            comments_count=Count('comments'),
            views_count=Count('views')
        )
    
    def optimized_for_list(self):
        """Optimize for list view"""
        return self.select_related('author').prefetch_related('tags').only(
            'id', 'title', 'excerpt', 'author__username', 'created_at'
        )
    
    def optimized_for_detail(self):
        """Optimize for detail view"""
        return self.select_related('author', 'category').prefetch_related(
            Prefetch('comments', queryset=Comment.objects.select_related('author')),
            'tags',
            'likes'
        )


class PostManager(models.Manager):
    """
    Custom manager using custom queryset
    """
    
    def get_queryset(self):
        return PostQuerySet(self.model, using=self._db)
    
    # Proxy methods for convenience
    def published(self):
        return self.get_queryset().published()
    
    def drafts(self):
        return self.get_queryset().drafts()
    
    def trending(self, days=7):
        return self.get_queryset().trending(days)
    
    def by_author(self, author):
        return self.get_queryset().by_author(author)


class Post(models.Model):
    # ... fields ...
    
    objects = PostManager()  # Custom manager
    
    class Meta:
        db_table = 'posts'
        indexes = [
            models.Index(fields=['status', '-published_at']),
            models.Index(fields=['author', '-created_at']),
        ]


# Usage - All chainable!
Post.objects.published()
Post.objects.published().by_author(user)
Post.objects.published().trending().with_stats()
Post.objects.published().optimized_for_list()

# In ViewSet:
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        if self.action == 'list':
            return Post.objects.published().optimized_for_list()
        elif self.action == 'retrieve':
            return Post.objects.published().optimized_for_detail()
        return Post.objects.all()
```

**When to use:**
- **Reusable queries**: Common query patterns
- **Business logic**: Encapsulate in model layer
- **Chainable**: Combine multiple filters
- **Performance**: Pre-optimized queries

---

#### Pattern 3: Model Properties & Methods

```python
class Post(TimeStampedModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    status = models.CharField(
        max_length=20,
        choices=[
            ('draft', 'Draft'),
            ('published', 'Published'),
            ('archived', 'Archived'),
        ],
        default='draft'
    )
    published_at = models.DateTimeField(null=True, blank=True)
    
    # Property: Computed value (no arguments)
    @property
    def is_published(self):
        """Check if post is published"""
        return (
            self.status == 'published' and
            self.published_at and
            self.published_at <= timezone.now()
        )
    
    @property
    def reading_time(self):
        """Calculate reading time in minutes"""
        word_count = len(self.content.split())
        return max(1, word_count // 200)
    
    @property
    def excerpt(self):
        """Get first 100 characters"""
        return self.content[:100] + ('...' if len(self.content) > 100 else '')
    
    # Method: Action with side effects
    def publish(self, commit=True):
        """Publish the post"""
        self.status = 'published'
        self.published_at = timezone.now()
        
        if commit:
            self.save(update_fields=['status', 'published_at'])
        
        return self
    
    def archive(self):
        """Archive the post"""
        self.status = 'archived'
        self.save(update_fields=['status'])
    
    def increment_views(self):
        """Atomic view count increment"""
        Post.objects.filter(pk=self.pk).update(
            views_count=F('views_count') + 1
        )
    
    # Class method: Alternative constructor
    @classmethod
    def create_with_tags(cls, title, content, author, tag_names):
        """Create post and associate tags"""
        post = cls.objects.create(
            title=title,
            content=content,
            author=author
        )
        
        for tag_name in tag_names:
            tag, _ = Tag.objects.get_or_create(name=tag_name)
            post.tags.add(tag)
        
        return post
    
    # Static method: Utility function
    @staticmethod
    def validate_content_length(content):
        """Validate content length"""
        word_count = len(content.split())
        return word_count >= 100


# Usage:
post = Post.objects.get(id=1)

# Properties (computed)
if post.is_published:
    print(f"Reading time: {post.reading_time} minutes")

# Methods (actions)
post.publish()
post.increment_views()

# Class methods
post = Post.create_with_tags(
    title="New Post",
    content="...",
    author=user,
    tag_names=['django', 'python']
)
```

**Design philosophy:**
- **Properties**: Computed values, no side effects, fast
- **Methods**: Actions with side effects, can be slow
- **Class methods**: Alternative constructors
- **Static methods**: Utility functions

---

#### Pattern 4: Model Signals (Event-Driven)

```python
from django.db.models.signals import post_save, pre_save, post_delete, pre_delete
from django.dispatch import receiver

# Pattern A: Simple signals
@receiver(post_save, sender=Post)
def post_saved(sender, instance, created, **kwargs):
    """
    Triggered AFTER post is saved
    
    created=True: New post created
    created=False: Existing post updated
    """
    if created:
        # New post created
        print(f"New post created: {instance.title}")
        
        # Send notification to followers
        notify_followers.delay(instance.author.id, instance.id)
        
        # Update author stats
        instance.author.post_count = F('post_count') + 1
        instance.author.save(update_fields=['post_count'])
    
    else:
        # Post updated
        print(f"Post updated: {instance.title}")
        
        # Clear cache
        cache.delete(f'post:{instance.id}')


@receiver(pre_save, sender=Post)
def post_presave(sender, instance, **kwargs):
    """
    Triggered BEFORE post is saved
    Good for: Validation, auto-filling fields
    """
    # Auto-generate slug from title
    if not instance.slug:
        instance.slug = slugify(instance.title)
    
    # Set published_at on first publish
    if instance.status == 'published' and not instance.published_at:
        instance.published_at = timezone.now()


@receiver(post_delete, sender=Post)
def post_deleted(sender, instance, **kwargs):
    """Triggered AFTER post is deleted"""
    # Clean up related data
    cache.delete(f'post:{instance.id}')
    
    # Update author stats
    instance.author.post_count = F('post_count') - 1
    instance.author.save(update_fields=['post_count'])


# Pattern B: Conditional signals
@receiver(post_save, sender=Post)
def notify_on_publish(sender, instance, created, **kwargs):
    """Only notify when post is published"""
    if not created:  # Only for updates
        # Check if status changed to published
        try:
            old_instance = Post.objects.get(pk=instance.pk)
            if old_instance.status != 'published' and instance.status == 'published':
                # Status changed to published!
                send_publish_notification.delay(instance.id)
        except Post.DoesNotExist:
            pass


# Pattern C: Signal with sender check
@receiver(post_save)
def universal_post_save(sender, instance, **kwargs):
    """
    Listen to ALL model saves
    Use with caution - can slow down system!
    """
    if hasattr(instance, 'author'):
        print(f"{sender.__name__} saved by {instance.author}")


# Pattern D: Disconnect signals temporarily
from django.db.models.signals import post_save

def bulk_create_posts(posts_data):
    """
    Create many posts without triggering signals
    Useful for data imports, migrations
    """
    # Disconnect signal
    post_save.disconnect(post_saved, sender=Post)
    
    try:
        # Bulk create without triggering signals
        posts = [Post(**data) for data in posts_data]
        Post.objects.bulk_create(posts)
    finally:
        # Reconnect signal
        post_save.connect(post_saved, sender=Post)


# Pattern E: Custom signals
from django.dispatch import Signal

# Define custom signal
post_viewed = Signal()  # providing_args=['post', 'user', 'ip']

# views.py
class PostViewSet(viewsets.ModelViewSet):
    def retrieve(self, request, pk=None):
        post = self.get_object()
        
        # Trigger custom signal
        post_viewed.send(
            sender=self.__class__,
            post=post,
            user=request.user,
            ip=request.META.get('REMOTE_ADDR')
        )
        
        return super().retrieve(request, pk=pk)

# signals.py
@receiver(post_viewed)
def track_post_view(sender, post, user, ip, **kwargs):
    """Track post views"""
    PostView.objects.create(
        post=post,
        user=user if user.is_authenticated else None,
        ip_address=ip
    )
    
    # Update view count
    post.increment_views()
```

**When to use signals:**
✅ **Good use cases:**
- Logging and auditing
- Sending notifications
- Cache invalidation
- Updating related data
- Analytics tracking

❌ **Avoid for:**
- Complex business logic (use services instead)
- Long-running tasks (use Celery)
- Operations that can fail (makes debugging hard)
- Tight coupling between apps

**Enterprise pattern:**

```python
# apps/posts/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from .models import Post
from .tasks import notify_followers, update_search_index

@receiver(post_save, sender=Post)
def post_saved_handler(sender, instance, created, **kwargs):
    """
    Central signal handler
    Delegates to async tasks for heavy operations
    """
    if created:
        # Async tasks for expensive operations
        notify_followers.delay(instance.id)
        update_search_index.delay(instance.id)
    else:
        # Sync cache invalidation (fast)
        cache.delete(f'post:{instance.id}')


# apps/posts/apps.py
from django.apps import AppConfig

class PostsConfig(AppConfig):
    name = 'apps.posts'
    
    def ready(self):
        """Import signals when app is ready"""
        import apps.posts.signals  # noqa
```

---

#### Pattern 5: Multi-Tenant Models

```python
from django.db import models

class Tenant(models.Model):
    """
    Multi-tenant pattern: Each tenant has isolated data
    """
    name = models.CharField(max_length=100)
    subdomain = models.CharField(max_length=50, unique=True)
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)


class TenantAwareModel(models.Model):
    """
    Base model for tenant-specific data
    """
    tenant = models.ForeignKey(
        Tenant,
        on_delete=models.CASCADE,
        db_index=True
    )
    
    class Meta:
        abstract = True
    
    def save(self, *args, **kwargs):
        """Ensure tenant is set"""
        if not self.tenant_id:
            raise ValueError("Tenant must be set")
        super().save(*args, **kwargs)


class TenantAwareManager(models.Manager):
    """
    Manager that automatically filters by current tenant
    """
    
    def __init__(self, *args, **kwargs):
        self.tenant_id = kwargs.pop('tenant_id', None)
        super().__init__(*args, **kwargs)
    
    def get_queryset(self):
        qs = super().get_queryset()
        
        # Filter by tenant from thread-local storage
        from .middleware import get_current_tenant
        tenant = get_current_tenant()
        
        if tenant:
            return qs.filter(tenant=tenant)
        
        return qs


class Post(TenantAwareModel):
    """
    Tenant-aware post model
    """
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    
    objects = TenantAwareManager()
    
    class Meta:
        db_table = 'posts'
        indexes = [
            # Composite index for tenant + common queries
            models.Index(fields=['tenant', '-created_at']),
            models.Index(fields=['tenant', 'author']),
        ]
        # Ensure data isolation
        unique_together = [['tenant', 'slug']]


# Middleware to set current tenant
import threading

_thread_locals = threading.local()

def get_current_tenant():
    return getattr(_thread_locals, 'tenant', None)

def set_current_tenant(tenant):
    _thread_locals.tenant = tenant

class TenantMiddleware:
    """
    Middleware to extract tenant from request
    """
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Extract tenant from subdomain
        host = request.get_host().split(':')[0]
        subdomain = host.split('.')[0]
        
        try:
            tenant = Tenant.objects.get(subdomain=subdomain, is_active=True)
            set_current_tenant(tenant)
            request.tenant = tenant
        except Tenant.DoesNotExist:
            set_current_tenant(None)
            request.tenant = None
        
        response = self.get_response(request)
        
        # Clean up
        set_current_tenant(None)
        
        return response


# ViewSet automatically filters by tenant
class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    
    def get_queryset(self):
        # Automatically filtered by tenant via manager
        return Post.objects.all()
    
    def perform_create(self, serializer):
        # Auto-assign tenant from request
        serializer.save(
            author=self.request.user,
            tenant=self.request.tenant
        )
```

**When to use:**
- **SaaS applications**: Multiple clients, isolated data
- **B2B platforms**: Each business has own data
- **White-label**: Same app, different branding
- **Enterprise**: Data isolation requirements

---

#### Pattern 6: Versioned Models (History Tracking)

```python
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType
import json

class Version(models.Model):
    """
    Track changes to any model
    """
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    content_object = GenericForeignKey('content_type', 'object_id')
    
    # Version data
    version_number = models.IntegerField()
    data = models.JSONField()  # Snapshot of object state
    changes = models.JSONField(null=True)  # What changed
    
    # Metadata
    created_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    created_at = models.DateTimeField(auto_now_add=True)
    comment = models.TextField(blank=True)
    
    class Meta:
        ordering = ['-version_number']
        indexes = [
            models.Index(fields=['content_type', 'object_id', '-version_number']),
        ]


class VersionedModel(models.Model):
    """
    Base model with automatic versioning
    """
    version_number = models.IntegerField(default=1)
    
    class Meta:
        abstract = True
    
    def save(self, *args, **kwargs):
        """Create version on save"""
        # Get old version if updating
        old_data = None
        if self.pk:
            try:
                old_instance = self.__class__.objects.get(pk=self.pk)
                old_data = model_to_dict(old_instance)
                self.version_number = old_instance.version_number + 1
            except self.__class__.DoesNotExist:
                pass
        
        # Save the model
        super().save(*args, **kwargs)
        
        # Create version record
        new_data = model_to_dict(self)
        
        changes = {}
        if old_data:
            for key, new_value in new_data.items():
                old_value = old_data.get(key)
                if old_value != new_value:
                    changes[key] = {
                        'old': old_value,
                        'new': new_value
                    }
        
        Version.objects.create(
            content_object=self,
            version_number=self.version_number,
            data=new_data,
            changes=changes if changes else None
        )
    
    def get_versions(self):
        """Get all versions of this object"""
        ct = ContentType.objects.get_for_model(self.__class__)
        return Version.objects.filter(
            content_type=ct,
            object_id=self.pk
        )
    
    def revert_to_version(self, version_number):
        """Revert to specific version"""
        version = self.get_versions().get(version_number=version_number)
        
        for field, value in version.data.items():
            if hasattr(self, field):
                setattr(self, field, value)
        
        self.save()


class Post(VersionedModel, TimeStampedModel):
    """
    Post with automatic version tracking
    """
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE)


# Usage:
post = Post.objects.get(id=1)
post.title = "Updated Title"
post.save()  # Auto-creates version

# View history
versions = post.get_versions()
for version in versions:
    print(f"Version {version.version_number}")
    print(f"Changed: {version.changes}")

# Revert
post.revert_to_version(3)
```

**When to use:**
- **Audit requirements**: Track all changes
- **Compliance**: Legal/regulatory needs
- **Undo functionality**: Let users revert changes
- **Collaboration**: See who changed what

---

#### Pattern 7: Cached Properties

```python
from django.utils.functional import cached_property
from django.core.cache import cache

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    
    # Pattern A#### Pattern 7: Cached Views (Performance)

```python
from django.core.cache import cache
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from rest_framework import viewsets
from rest_framework.response import Response
import hashlib
import json

class CachedPostViewSet(viewsets.ModelViewSet):
    """
    Enterprise caching patterns for high-traffic APIs
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    
    # Pattern A: Simple time-based caching
    @method_decorator(cache_page(60 * 15))  # 15 minutes
    def list(self, request):
        """
        Cache entire response for 15 minutes
        Good for: Public data, infrequent changes
        """
        return super().list(request)
    
    # Pattern B: Custom cache key with query params
    def retrieve(self, request, pk=None):
        """
        Cache with custom key including query parameters
        """
        # Create cache key from URL + query params
        query_string = request.GET.urlencode()
        cache_key = f'post:{pk}:{hashlib.md5(query_string.encode()).hexdigest()}'
        
        # Try cache first
        cached_data = cache.get(cache_key)
        if cached_data:
            return Response(cached_data)
        
        # Cache miss - fetch from database
        response = super().retrieve(request, pk=pk)
        
        # Cache for 1 hour
        cache.set(cache_key, response.data, 60 * 60)
        
        return response
    
    # Pattern C: Cache invalidation on update
    def update(self, request, pk=None):
        """Invalidate cache when data changes"""
        # Perform update
        response = super().update(request, pk=pk)
        
        # Invalidate related caches
        self._invalidate_post_cache(pk)
        
        return response
    
    def destroy(self, request, pk=None):
        """Invalidate cache when deleted"""
        response = super().destroy(request, pk=pk)
        self._invalidate_post_cache(pk)
        return response
    
    def _invalidate_post_cache(self, post_id):
        """Helper to invalidate all cached versions of a post"""
        # Delete specific post cache
        cache.delete(f'post:{post_id}')
        
        # Delete list cache (contains this post)
        cache.delete('post_list')
        
        # Delete user's posts cache
        post = Post.objects.get(id=post_id)
        cache.delete(f'user:{post.author_id}:posts')


# Pattern D: Smart caching with ETags
class SmartCachedViewSet(viewsets.ModelViewSet):
    """
    Use ETags for conditional requests
    Client sends If-None-Match, server returns 304 if unchanged
    """
    
    def retrieve(self, request, pk=None):
        post = self.get_object()
        
        # Generate ETag from updated_at timestamp
        etag = f'"{post.updated_at.timestamp()}"'
        
        # Check If-None-Match header
        if request.META.get('HTTP_IF_NONE_MATCH') == etag:
            return Response(status=304)  # Not Modified
        
        serializer = self.get_serializer(post)
        response = Response(serializer.data)
        response['ETag'] = etag
        return response


# Pattern E: Multi-level caching (Redis + Memory)
from django.core.cache import caches

class MultiLevelCacheViewSet(viewsets.ModelViewSet):
    """
    L1 Cache: In-memory (fastest, small)
    L2 Cache: Redis (fast, larger)
    L3 Cache: Database (slowest, complete)
    """
    
    def retrieve(self, request, pk=None):
        cache_key = f'post:{pk}'
        
        # Try L1 (in-memory)
        local_cache = caches['local']
        data = local_cache.get(cache_key)
        if data:
            return Response(data)
        
        # Try L2 (Redis)
        redis_cache = caches['default']
        data = redis_cache.get(cache_key)
        if data:
            # Populate L1 for next request
            local_cache.set(cache_key, data, 60)
            return Response(data)
        
        # L3: Fetch from database
        post = self.get_object()
        serializer = self.get_serializer(post)
        data = serializer.data
        
        # Populate caches
        redis_cache.set(cache_key, data, 60 * 60)  # 1 hour
        local_cache.set(cache_key, data, 60)       # 1 minute
        
        return Response(data)
```

**Cache configuration:**

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'TIMEOUT': 3600,  # 1 hour default
    },
    'local': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'LOCATION': 'unique-snowflake',
        'TIMEOUT': 60,  # 1 minute
    }
}
```

**When to use caching:**
- **High traffic**: Same data requested frequently
- **Expensive queries**: Complex aggregations, joins
- **External APIs**: Rate-limited third-party calls
- **Read-heavy**: 90%+ reads, few writes

---

#### Pattern 8: Real-time Views (WebSocket Integration)

```python
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync
from rest_framework import viewsets
from rest_framework.decorators import action

class RealtimePostViewSet(viewsets.ModelViewSet):
    """
    Integrate with Django Channels for real-time updates
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    
    def create(self, request):
        """Create post and broadcast to WebSocket clients"""
        response = super().create(request)
        
        # Broadcast to WebSocket
        channel_layer = get_channel_layer()
        async_to_sync(channel_layer.group_send)(
            'posts',  # Group name
            {
                'type': 'post.created',
                'post': response.data
            }
        )
        
        return response
    
    def update(self, request, pk=None):
        """Update post and notify subscribers"""
        response = super().update(request, pk=pk)
        
        channel_layer = get_channel_layer()
        async_to_sync(channel_layer.group_send)(
            f'post.{pk}',  # Specific post channel
            {
                'type': 'post.updated',
                'post': response.data
            }
        )
        
        return response
    
    @action(detail=True, methods=['post'])
    def like(self, request, pk=None):
        """Real-time like counter updates"""
        post = self.get_object()
        Like.objects.get_or_create(user=request.user, post=post)
        
        # Get updated like count
        like_count = post.likes.count()
        
        # Broadcast to all viewing this post
        channel_layer = get_channel_layer()
        async_to_sync(channel_layer.group_send)(
            f'post.{pk}',
            {
                'type': 'post.like_count_updated',
                'post_id': pk,
                'like_count': like_count
            }
        )
        
        return Response({'like_count': like_count})


# WebSocket Consumer (consumers.py)
from channels.generic.websocket import AsyncJsonWebsocketConsumer

class PostConsumer(AsyncJsonWebsocketConsumer):
    async def connect(self):
        self.post_id = self.scope['url_route']['kwargs']['post_id']
        self.room_group_name = f'post.{self.post_id}'
        
        # Join room group
        await self.channel_layer.group_add(
            self.room_group_name,
            self.channel_name
        )
        
        await self.accept()
    
    async def disconnect(self, close_code):
        # Leave room group
        await self.channel_layer.group_discard(
            self.room_group_name,
            self.channel_name
        )
    
    # Receive message from WebSocket
    async def receive_json(self, content):
        pass
    
    # Receive message from room group
    async def post_updated(self, event):
        # Send message to WebSocket
        await self.send_json({
            'type': 'post_updated',
            'post': event['post']
        })
    
    async def post_like_count_updated(self, event):
        await self.send_json({
            'type': 'like_count_updated',
            'post_id': event['post_id'],
            'like_count': event['like_count']
        })
```

**When to use real-time:**
- **Collaborative editing**: Multiple users editing same document
- **Live updates**: Comments, likes, status changes
- **Notifications**: Push notifications to users
- **Dashboards**: Real-time analytics, monitoring

---

### Deep Dive: Serializer Patterns - 10 Different Approaches

#### Pattern 1: Basic ModelSerializer

```python
class PostSerializer(serializers.ModelSerializer):
    """
    Most common pattern: Auto-generate fields from model
    
    Execution:
    1. Introspects Post model
    2. Creates fields based on model fields
    3. Handles validation automatically
    """
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'created_at']
        read_only_fields = ['id', 'created_at']
        
    # Automatic features:
    # - Field types inferred from model
    # - Validators from model constraints
    # - Default values from model
    # - Relationships handled automatically
```

**When to use:**
- **Standard CRUD**: Straightforward model representation
- **Quick prototypes**: Fast development
- **80% of cases**: Sufficient for most needs

---

#### Pattern 2: Explicit Field Declaration

```python
class PostSerializer(serializers.Serializer):
    """
    Manual field declaration - full control
    More verbose but explicit
    """
    
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    content = serializers.CharField(style={'base_template': 'textarea.html'})
    author_id = serializers.IntegerField()
    published = serializers.BooleanField(default=False)
    created_at = serializers.DateTimeField(read_only=True)
    
    def validate_title(self, value):
        """Field-level validation"""
        if len(value) < 5:
            raise serializers.ValidationError("Title too short")
        return value
    
    def validate(self, data):
        """Object-level validation"""
        if data.get('published') and not data.get('content'):
            raise serializers.ValidationError(
                "Cannot publish post without content"
            )
        return data
    
    def create(self, validated_data):
        """Must implement create manually"""
        return Post.objects.create(**validated_data)
    
    def update(self, instance, validated_data):
        """Must implement update manually"""
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.published = validated_data.get('published', instance.published)
        instance.save()
        return instance
```

**When to use:**
- **Non-model data**: Data not backed by Django models
- **Custom logic**: Need full control over serialization
- **External APIs**: Serializing third-party API responses
- **Learning**: Understanding how serializers work

---

#### Pattern 3: Nested Serializers (Read-only)

```python
class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'profile_picture']

class CommentSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    
    class Meta:
        model = Comment
        fields = ['id', 'content', 'author', 'created_at']

class PostDetailSerializer(serializers.ModelSerializer):
    """
    Nested serializers for rich representation
    READ-ONLY: Can't create/update nested objects
    """
    author = AuthorSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    tags = serializers.StringRelatedField(many=True, read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'comments', 'tags', 'created_at']

# Output:
{
    "id": 1,
    "title": "My Post",
    "content": "...",
    "author": {
        "id": 5,
        "username": "john",
        "email": "john@example.com",
        "profile_picture": "https://..."
    },
    "comments": [
        {
            "id": 10,
            "content": "Great post!",
            "author": {"id": 6, "username": "jane", ...},
            "created_at": "2025-10-06T10:00:00Z"
        }
    ],
    "tags": ["django", "python", "api"]
}
```

**When to use:**
- **Detail views**: Rich representation with related data
- **Read-only**: Displaying data, not accepting input
- **API clients**: Reduce number of requests needed

**Performance consideration:**

```python
# ⚠️ WARNING: This causes N+1 queries!
class PostDetailSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)  # Query for each post!
    comments = CommentSerializer(many=True, read_only=True)  # N queries!

# ✅ SOLUTION: Optimize queryset in view
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        queryset = super().get_queryset()
        
        if self.action == 'retrieve':
            queryset = queryset.select_related('author').prefetch_related(
                'comments', 'comments__author', 'tags'
            )
        
        return queryset
```

---

#### Pattern 4: Writable Nested Serializers

```python
class CommentCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Comment
        fields = ['content', 'author_id']

class PostCreateSerializer(serializers.ModelSerializer):
    """
    Accept nested data for creation
    More complex: must handle creation of related objects
    """
    comments = CommentCreateSerializer(many=True, required=False)
    tags = serializers.ListField(
        child=serializers.CharField(max_length=50),
        required=False
    )
    
    class Meta:
        model = Post
        fields = ['title', 'content', 'comments', 'tags']
    
    def create(self, validated_data):
        """Handle nested creation"""
        comments_data = validated_data.pop('comments', [])
        tags_data = validated_data.pop('tags', [])
        
        # Create post
        post = Post.objects.create(**validated_data)
        
        # Create comments
        for comment_data in comments_data:
            Comment.objects.create(post=post, **comment_data)
        
        # Create/get tags
        for tag_name in tags_data:
            tag, created = Tag.objects.get_or_create(name=tag_name)
            post.tags.add(tag)
        
        return post
    
    def update(self, instance, validated_data):
        """Handle nested updates"""
        comments_data = validated_data.pop('comments', None)
        tags_data = validated_data.pop('tags', None)
        
        # Update post fields
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.save()
        
        # Update comments if provided
        if comments_data is not None:
            # Delete existing comments
            instance.comments.all().delete()
            
            # Create new comments
            for comment_data in comments_data:
                Comment.objects.create(post=instance, **comment_data)
        
        # Update tags if provided
        if tags_data is not None:
            instance.tags.clear()
            for tag_name in tags_data:
                tag, created = Tag.objects.get_or_create(name=tag_name)
                instance.tags.add(tag)
        
        return instance


# Request example:
POST /api/posts/
{
    "title": "My Post",
    "content": "Post content",
    "comments": [
        {"content": "First comment", "author_id": 5},
        {"content": "Second comment", "author_id": 6}
    ],
    "tags": ["django", "python"]
}
```

**When to use:**
- **Complex creation**: Create post with comments in one request
- **Atomic operations**: Related objects created together
- **Better UX**: Single API call instead of multiple

**Trade-offs:**
- ✅ **Pros**: Fewer API calls, atomic, better UX
- ❌ **Cons**: Complex code, harder validation, error handling

---

#### Pattern 5: Dynamic Field Serializers

```python
class DynamicFieldsModelSerializer(serializers.ModelSerializer):
    """
    A ModelSerializer that takes an additional `fields` argument
    to dynamically include/exclude fields
    
    Usage: GET /api/posts/?fields=id,title,author
    """
    
    def __init__(self, *args, **kwargs):
        # Extract fields from context or kwargs
        fields = kwargs.pop('fields', None)
        
        # Optionally get fields from request
        request = kwargs.get('context', {}).get('request')
        if request and not fields:
            fields = request.query_params.get('fields')
        
        super().__init__(*args, **kwargs)
        
        if fields:
            fields = fields.split(',') if isinstance(fields, str) else fields
            # Drop any fields not specified
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)


class PostSerializer(DynamicFieldsModelSerializer):
    author = UserSerializer(read_only=True)
    comments_count = serializers.IntegerField(read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'comments_count', 'created_at']


# Usage in view:
class PostViewSet(viewsets.ModelViewSet):
    serializer_class = PostSerializer
    
    def get_serializer(self, *args, **kwargs):
        """Pass fields from query params to serializer"""
        serializer_class = self.get_serializer_class()
        kwargs['context'] = self.get_serializer_context()
        return serializer_class(*args, **kwargs)


# API Calls:
GET /api/posts/?fields=id,title
# Response: [{"id": 1, "title": "Post 1"}, ...]

GET /api/posts/?fields=id,title,author
# Response: [{"id": 1, "title": "Post 1", "author": {...}}, ...]
```

**When to use:**
- **Flexible APIs**: Clients need different field sets
- **Mobile apps**: Reduce bandwidth for limited data
- **GraphQL-like**: Similar to GraphQL field selection
- **Performance**: Only fetch/serialize needed fields

---

#### Pattern 6: Multiple Serializers per Model

```python
class PostListSerializer(serializers.ModelSerializer):
    """
    Minimal serializer for list view
    Fast serialization, minimal data
    """
    author_name = serializers.CharField(source='author.username', read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'author_name', 'created_at']


class PostDetailSerializer(serializers.ModelSerializer):
    """
    Complete serializer for detail view
    All fields and relationships
    """
    author = UserSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    tags = TagSerializer(many=True, read_only=True)
    likes_count = serializers.IntegerField(read_only=True)
    is_liked = serializers.SerializerMethodField()
    
    class Meta:
        model = Post
        fields = [
            'id', 'title', 'content', 'author', 'comments', 
            'tags', 'likes_count', 'is_liked', 'created_at', 'updated_at'
        ]
    
    def get_is_liked(self, obj):
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.likes.filter(user=request.user).exists()
        return False


class PostCreateSerializer(serializers.ModelSerializer):
    """
    Writable serializer for creation
    Only accepts writable fields
    """
    tag_ids = serializers.ListField(
        child=serializers.IntegerField(),
        write_only=True,
        required=False
    )
    
    class Meta:
        model = Post
        fields = ['title', 'content', 'tag_ids']
    
    def create(self, validated_data):
        tag_ids = validated_data.pop('tag_ids', [])
        post = Post.objects.create(**validated_data)
        
        if tag_ids:
            post.tags.set(tag_ids)
        
        return post


class PostUpdateSerializer(serializers.ModelSerializer):
    """
    Update serializer with different validation rules
    """
    class Meta:
        model = Post
        fields = ['title', 'content', 'published']
    
    def validate_published(self, value):
        """Can't unpublish a post"""
        if self.instance and self.instance.published and not value:
            raise serializers.ValidationError(
                "Cannot unpublish a post"
            )
        return value


# Use in ViewSet
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    
    def get_serializer_class(self):
        """Return different serializer based on action"""
        if self.action == 'list':
            return PostListSerializer
        elif self.action == 'retrieve':
            return PostDetailSerializer
        elif self.action == 'create':
            return PostCreateSerializer
        elif self.action in ['update', 'partial_update']:
            return PostUpdateSerializer
        return PostSerializer  # Default
    
    def get_queryset(self):
        """Optimize queryset based on serializer needs"""
        queryset = super().get_queryset()
        
        if self.action == 'list':
            # Minimal fields for list serializer
            queryset = queryset.select_related('author').only(
                'id', 'title', 'author__username', 'created_at'
            )
        elif self.action == 'retrieve':
            # All fields for detail serializer
            queryset = queryset.select_related('author').prefetch_related(
                'comments', 'comments__author', 'tags', 'likes'
            ).annotate(likes_count=Count('likes'))
        
        return queryset
```

**When to use:**
- **Different contexts**: List vs detail vs create
- **Performance**: Optimize for each use case
- **Validation**: Different rules for create vs update
- **Enterprise**: Professional API design

---

#### Pattern 7: SerializerMethodField (Computed Fields)

```python
class PostSerializer(serializers.ModelSerializer):
    """
    Use SerializerMethodField for computed values
    """
    
    # Computed fields
    reading_time = serializers.SerializerMethodField()
    is_trending = serializers.SerializerMethodField()
    author_info = serializers.SerializerMethodField()
    relative_time = serializers.SerializerMethodField()
    
    class Meta:
        model = Post
        fields = [
            'id', 'title', 'content',
            'reading_time', 'is_trending', 'author_info', 'relative_time'
        ]
    
    def get_reading_time(self, obj):
        """
        Calculate reading time based on word count
        Executes for EACH object during serialization
        """
        word_count = len(obj.content.split())
        minutes = word_count // 200  # Average reading speed
        return f"{minutes} min read" if minutes > 0 else "< 1 min read"
    
    def get_is_trending(self, obj):
        """
        Check if post is trending
        ⚠️ WARNING: This can cause N+1 queries!
        """
        # Bad: Queries database for each post
        # recent_likes = obj.likes.filter(
        #     created_at__gte=timezone.now() - timedelta(days=1)
        # ).count()
        
        # Good: Use annotated value from queryset
        return getattr(obj, 'is_trending', False)
    
    def get_author_info(self, obj):
        """
        Return custom author information
        Uses select_related to avoid N+1
        """
        return {
            'id': obj.author.id,
            'username': obj.author.username,
            'is_verified': obj.author.is_verified,
            'posts_count': getattr(obj.author, 'posts_count', 0)
        }
    
    def get_relative_time(self, obj):
        """
        Human-readable relative time
        """
        from django.utils.timesince import timesince
        return f"{timesince(obj.created_at)} ago"


# Optimize in ViewSet
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return Post.objects.select_related('author').annotate(
            # Pre-compute trending status
            is_trending=Exists(
                Like.objects.filter(
                    post=OuterRef('pk'),
                    created_at__gte=timezone.now() - timedelta(days=1)
                ).values('pk')[:10]
            )
        )
```

**When to use:**
- **Computed values**: Not stored in database
- **Custom logic**: Complex calculations
- **Context-dependent**: Value depends on request user

**Performance tips:**
```python
# ❌ BAD: Causes queries
def get_comments_count(self, obj):
    return obj.comments.count()  # Query per object!

# ✅ GOOD: Use annotation
# In ViewSet: queryset.annotate(comments_count=Count('comments'))
comments_count = serializers.IntegerField(read_only=True)

# ❌ BAD: Heavy computation
def get_similarity_score(self, obj):
    # Complex ML calculation for each object
    return calculate_similarity(obj)

# ✅ GOOD: Pre-compute and store
# Add 'similarity_score' field to model, compute async
similarity_score = serializers.FloatField(read_only=True)
```

---

#### Pattern 8: Conditional Field Inclusion

```python
class ConditionalPostSerializer(serializers.ModelSerializer):
    """
    Include/exclude fields based on conditions
    """
    
    # Optional fields
    internal_notes = serializers.CharField(read_only=True)
    draft_content = serializers.CharField(read_only=True)
    author_email = serializers.EmailField(source='author.email', read_only=True)
    
    class Meta:
        model = Post
        fields = [
            'id', 'title', 'content',
            'internal_notes', 'draft_content', 'author_email'
        ]
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
        request = self.context.get('request')
        
        # Remove sensitive fields for non-staff users
        if request and not request.user.is_staff:
            self.fields.pop('internal_notes', None)
            self.fields.pop('draft_content', None)
            self.fields.pop('author_email', None)
        
        # Remove fields based on user permissions
        if request and not request.user.has_perm('posts.view_draft'):
            self.fields.pop('draft_content', None)


class PermissionAwareSerializer(serializers.ModelSerializer):
    """
    Advanced: Field-level permissions
    """
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
        request = self.context.get('request')
        if not request:
            return
        
        # Define field permissions
        field_permissions = {
            'salary': 'view_salary',
            'ssn': 'view_ssn',
            'internal_notes': 'view_internal',
        }
        
        # Remove fields user doesn't have permission for
        for field_name, permission in field_permissions.items():
            if not request.user.has_perm(f'posts.{permission}'):
                self.fields.pop(field_name, None)
```

**When to use:**
- **Role-based**: Different fields for different roles
- **Privacy**: Hide sensitive data
- **Permissions**: Field-level access control
- **Multi-tenant**: Different data per tenant

---

#### Pattern 9: Validation Patterns

```python
class AdvancedValidationSerializer(serializers.ModelSerializer):
    """
    Comprehensive validation patterns
    """
    
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    publish_date = serializers.DateTimeField(required=False)
    category = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all()
    )
    
    class Meta:
        model = Post
        fields = ['title', 'content', 'publish_date', 'category']
    
    # Pattern A: Field-level validation
    def validate_title(self, value):
        """
        Validate single field
        Runs during is_valid()
        """
        if len(value) < 10:
            raise serializers.ValidationError(
                "Title must be at least 10 characters"
            )
        
        # Check for profanity
        if contains_profanity(value):
            raise serializers.ValidationError(
                "Title contains inappropriate content"
            )
        
        # Check uniqueness with custom message
        if Post.objects.filter(title__iexact=value).exists():
            raise serializers.ValidationError(
                "A post with this title already exists"
            )
        
        return value.strip()  # Clean and return
    
    def validate_publish_date(self, value):
        """Validate publish date is in future"""
        if value and value < timezone.now():
            raise serializers.ValidationError(
                "Publish date must be in the future"
            )
        return value
    
    # Pattern B: Object-level validation
    def validate(self, data):
        """
        Validate multiple fields together
        Runs after all field validations pass
        """
        # Cross-field validation
        if data.get('publish_date') and not data.get('content'):
            raise serializers.ValidationError(
                "Cannot schedule empty post"
            )
        
        # Business rule validation
        if dataRemember: **Start simple, measure performance, optimize when needed**.

---

## Enterprise-Grade Django REST Framework Patterns

### Deep Dive: URL Patterns - 7 Different Approaches

#### Approach 1: Simple Router (Basic CRUD)

```python
# urls.py
from rest_framework.routers import SimpleRouter
from posts.views import PostViewSet

router = SimpleRouter()
router.register(r'posts', PostViewSet, basename='post')

urlpatterns = router.urls

# Generated URLs:
# /posts/          → list, create
# /posts/{pk}/     → retrieve, update, partial_update, destroy
```

**When to use:**
- Simple APIs with standard CRUD
- No need for API root view
- Microservices with single resource

**Question:** *Why no trailing slash option?*
**Answer:** SimpleRouter is opinionated. Use DefaultRouter for trailing slash control.

---

#### Approach 2: DefaultRouter (API Root + Options)

```python
# urls.py
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'posts', PostViewSet)
router.register(r'users', UserViewSet)
router.register(r'comments', CommentViewSet)

urlpatterns = router.urls

# Generated URLs:
# /                    → API root (lists all endpoints)
# /posts/              → PostViewSet
# /users/              → UserViewSet
# /comments/           → CommentViewSet
# Each with OPTIONS method support
```

**When to use:**
- Public APIs that need discoverability
- Multiple resources
- Browsable API with root endpoint
- Need OPTIONS support for CORS

**Question:** *What's the difference from SimpleRouter?*
**Answer:** DefaultRouter adds:
1. API root view at `/`
2. Trailing slash support
3. `.json` format suffix
4. Better for production APIs

---

#### Approach 3: Nested Routers (Parent-Child Resources)

```python
# Install: pip install drf-nested-routers

from rest_framework_nested import routers

router = routers.DefaultRouter()
router.register(r'posts', PostViewSet, basename='post')

# Nested router for post comments
posts_router = routers.NestedDefaultRouter(router, r'posts', lookup='post')
posts_router.register(r'comments', CommentViewSet, basename='post-comments')

urlpatterns = [
    path('api/', include(router.urls)),
    path('api/', include(posts_router.urls)),
]

# Generated URLs:
# /api/posts/                          → All posts
# /api/posts/{post_pk}/                → Single post
# /api/posts/{post_pk}/comments/       → Comments for post
# /api/posts/{post_pk}/comments/{pk}/  → Single comment
```

**ViewSet implementation:**

```python
class CommentViewSet(viewsets.ModelViewSet):
    serializer_class = CommentSerializer
    
    def get_queryset(self):
        # Access parent post_pk from URL
        post_pk = self.kwargs['post_pk']
        return Comment.objects.filter(post_id=post_pk)
    
    def perform_create(self, serializer):
        post_pk = self.kwargs['post_pk']
        serializer.save(post_id=post_pk, author=self.request.user)
```

**When to use:**
- Parent-child relationships (post → comments)
- RESTful resource nesting
- Clear hierarchical structure
- Better than: `/comments/?post_id=1`

**Question:** *Why not just use query parameters?*
**Answer:** 
- **Nested URLs**: Semantic, RESTful, clearer intent
- **Query params**: More flexible, better for filtering
- **Enterprise**: Use nested for core relationships, query params for filters

---

#### Approach 4: Manual URL Configuration (Full Control)

```python
# urls.py
from django.urls import path
from posts.views import (
    PostListCreateView,
    PostDetailView,
    PostPublishView,
    PostStatsView
)

urlpatterns = [
    path('posts/', PostListCreateView.as_view(), name='post-list'),
    path('posts/<int:pk>/', PostDetailView.as_view(), name='post-detail'),
    path('posts/<int:pk>/publish/', PostPublishView.as_view(), name='post-publish'),
    path('posts/stats/', PostStatsView.as_view(), name='post-stats'),
]

# Views
from rest_framework import generics

class PostListCreateView(generics.ListCreateAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

class PostDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

class PostPublishView(generics.GenericAPIView):
    queryset = Post.objects.all()
    
    def post(self, request, pk):
        post = self.get_object()
        post.published = True
        post.save()
        return Response({'status': 'published'})
```

**When to use:**
- Non-standard REST patterns
- Mixed ViewSet and APIView endpoints
- Custom URL patterns
- Fine-grained control needed

**Question:** *Isn't this more work than routers?*
**Answer:** Yes, but gives you:
- Custom URL patterns (`/posts/trending/`, `/posts/by-date/2024-01-01/`)
- Mix different view types
- No "magic" - explicit and clear
- Better for complex APIs

---

#### Approach 5: Versioned URLs (API Versioning)

```python
# urls.py - Approach A: URL Path Versioning
urlpatterns = [
    path('api/v1/', include('api.v1.urls')),
    path('api/v2/', include('api.v2.urls')),
]

# api/v1/urls.py
from rest_framework.routers import DefaultRouter
from .views import PostViewSetV1

router = DefaultRouter()
router.register(r'posts', PostViewSetV1)
urlpatterns = router.urls

# api/v2/urls.py
from rest_framework.routers import DefaultRouter
from .views import PostViewSetV2

router = DefaultRouter()
router.register(r'posts', PostViewSetV2)
urlpatterns = router.urls


# Approach B: Accept Header Versioning
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.AcceptHeaderVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
}

# Single URL, version in Accept header
# Request: Accept: application/json; version=v2

# urls.py
router = DefaultRouter()
router.register(r'posts', PostViewSet)

# views.py
class PostViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        if self.request.version == 'v2':
            return PostSerializerV2
        return PostSerializerV1


# Approach C: Query Parameter Versioning
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.QueryParameterVersioning',
}

# Request: GET /api/posts/?version=v2
```

**Comparison:**

| Method | Example | Pros | Cons | Enterprise Use |
|--------|---------|------|------|----------------|
| URL Path | `/api/v1/posts/` | Clear, cacheable | URL proliferation | ✅ Best for public APIs |
| Accept Header | `Accept: version=v2` | Clean URLs | Not browser-friendly | ✅ Internal services |
| Query Param | `?version=v2` | Simple | Breaks caching | ⚠️ Quick prototypes only |

**When to use:**
- Breaking changes in API
- Long-term API support
- Multiple client versions
- Enterprise requirement

---

#### Approach 6: Namespace URLs (Multi-tenant / Multi-app)

```python
# Main urls.py
urlpatterns = [
    path('api/blog/', include('blog.urls', namespace='blog')),
    path('api/shop/', include('shop.urls', namespace='shop')),
    path('api/forum/', include('forum.urls', namespace='forum')),
]

# blog/urls.py
app_name = 'blog'

router = DefaultRouter()
router.register(r'posts', PostViewSet, basename='post')

urlpatterns = [
    path('', include(router.urls)),
]

# Usage in code:
from django.urls import reverse

url = reverse('blog:post-detail', kwargs={'pk': 1})
# Result: /api/blog/posts/1/

# In serializers
class PostSerializer(serializers.HyperlinkedModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='blog:post-detail'
    )
```

**When to use:**
- Multi-tenant applications
- Separate Django apps
- Prevent URL name collisions
- Large monolithic projects

---

#### Approach 7: Dynamic URL Generation (Enterprise Pattern)

```python
# config/urls.py - Central configuration
from django.apps import apps
from rest_framework.routers import DefaultRouter

def generate_api_urls():
    """
    Automatically register all ViewSets from installed apps
    Enterprise pattern for large projects
    """
    router = DefaultRouter()
    
    for app_config in apps.get_app_configs():
        # Look for viewsets.py in each app
        try:
            viewsets_module = __import__(
                f'{app_config.name}.viewsets',
                fromlist=['']
            )
            
            # Auto-register viewsets with metadata
            for name in dir(viewsets_module):
                obj = getattr(viewsets_module, name)
                
                if (isinstance(obj, type) and 
                    issubclass(obj, viewsets.ModelViewSet) and
                    hasattr(obj, 'Meta')):
                    
                    prefix = obj.Meta.url_prefix
                    router.register(prefix, obj, basename=obj.Meta.basename)
        
        except (ImportError, AttributeError):
            continue
    
    return router.urls

urlpatterns = [
    path('api/', include(generate_api_urls())),
]

# posts/viewsets.py
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    
    class Meta:
        url_prefix = 'posts'
        basename = 'post'
```

**When to use:**
- Large enterprise applications (50+ apps)
- Plugin-based architecture
- Microservices monolith (modular monolith)
- Auto-discovery needed

**Question:** *Isn't auto-discovery "magic" and hard to debug?*
**Answer:** Yes! Trade-offs:
- **Pros**: DRY, scales to 100+ apps, plugin architecture
- **Cons**: Harder to debug, implicit behavior
- **Solution**: Use for stable enterprise apps with good documentation

---

### Deep Dive: View Patterns - 8 Different Approaches

#### Pattern 1: Function-Based Views (Simplest)

```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework import status

@api_view(['GET', 'POST'])
@permission_classes([IsAuthenticated])
def post_list(request):
    """
    Handle list and create operations
    
    Execution flow:
    1. @api_view wraps function
    2. Checks HTTP method against allowed methods
    3. Converts Django request → DRF Request
    4. Runs permission checks
    5. Executes function body
    6. Wraps return → DRF Response
    """
    
    if request.method == 'GET':
        posts = Post.objects.all()
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = PostSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


@api_view(['GET'])
def post_stats(request):
    """
    Custom endpoint: GET /api/posts/stats/
    """
    stats = {
        'total': Post.objects.count(),
        'published': Post.objects.filter(published=True).count(),
        'drafts': Post.objects.filter(published=False).count(),
        'by_author': list(
            Post.objects.values('author__username')
            .annotate(count=Count('id'))
        )
    }
    return Response(stats)
```

**When to use:**
- **Quick prototypes**: Fast to write
- **One-off endpoints**: Doesn't fit CRUD
- **Simple logic**: No complex state management
- **Utility endpoints**: Stats, exports, triggers

**Enterprise usage:**
```python
# Good for:
@api_view(['POST'])
def trigger_backup(request):
    backup_database.delay()  # Celery task
    return Response({'status': 'started'})

@api_view(['GET'])
def health_check(request):
    return Response({
        'status': 'healthy',
        'database': check_database(),
        'cache': check_cache(),
        'queue': check_queue()
    })
```

---

#### Pattern 2: APIView (Class-Based, Full Control)

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status, permissions

class PostList(APIView):
    """
    Class-based view with explicit method handlers
    
    Execution flow:
    1. URL dispatcher → PostList.as_view()
    2. as_view() returns view function
    3. Calls dispatch(request, *args, **kwargs)
    4. dispatch() → initial(request)
        - Runs authentication
        - Runs permissions
        - Runs throttling
    5. dispatch() → handler method (get/post/put/delete)
    6. Returns Response
    """
    
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    
    def get(self, request, format=None):
        """Handle GET /api/posts/"""
        posts = Post.objects.all()
        
        # Custom filtering logic
        status_filter = request.query_params.get('status')
        if status_filter:
            posts = posts.filter(status=status_filter)
        
        serializer = PostSerializer(posts, many=True, context={'request': request})
        return Response(serializer.data)
    
    def post(self, request, format=None):
        """Handle POST /api/posts/"""
        serializer = PostSerializer(data=request.data)
        
        if serializer.is_valid():
            # Custom business logic before save
            if request.user.post_count >= 10:
                return Response(
                    {'error': 'Post limit reached'},
                    status=status.HTTP_403_FORBIDDEN
                )
            
            serializer.save(author=request.user)
            
            # Custom business logic after save
            request.user.post_count += 1
            request.user.save()
            
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class PostDetail(APIView):
    """Handle single post operations"""
    
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    
    def get_object(self, pk):
        try:
            return Post.objects.get(pk=pk)
        except Post.DoesNotExist:
            raise Http404
    
    def get(self, request, pk, format=None):
        post = self.get_object(pk)
        serializer = PostSerializer(post, context={'request': request})
        return Response(serializer.data)
    
    def put(self, request, pk, format=None):
        post = self.get_object(pk)
        
        # Custom permission check
        if post.author != request.user:
            return Response(
                {'error': 'Not authorized'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        serializer = PostSerializer(post, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def delete(self, request, pk, format=None):
        post = self.get_object(pk)
        
        if post.author != request.user:
            return Response(
                {'error': 'Not authorized'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        post.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**When to use:**
- **Custom logic**: Business rules don't fit generic patterns
- **Non-CRUD**: Complex operations (multi-step workflows)
- **Mixed operations**: Different serializers per method
- **Fine control**: Need to customize every step

**Enterprise pattern:**

```python
class PaymentProcessView(APIView):
    """
    Complex multi-step operation
    Not CRUD, so APIView is perfect
    """
    permission_classes = [IsAuthenticated]
    
    def post(self, request):
        # Step 1: Validate payment data
        serializer = PaymentSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        
        with transaction.atomic():
            # Step 2: Create payment record
            payment = serializer.save(user=request.user)
            
            # Step 3: Call payment gateway
            gateway_response = stripe.charge(
                amount=payment.amount,
                source=request.data['token']
            )
            
            # Step 4: Update payment status
            payment.gateway_id = gateway_response['id']
            payment.status = 'completed'
            payment.save()
            
            # Step 5: Create order
            order = Order.objects.create(
                user=request.user,
                payment=payment,
                items=request.data['items']
            )
            
            # Step 6: Send confirmation
            send_confirmation_email.delay(order.id)
        
        return Response({
            'payment_id': payment.id,
            'order_id': order.id,
            'status': 'success'
        }, status=status.HTTP_201_CREATED)
```

---

#### Pattern 3: Generic Views (Pre-built Patterns)

```python
from rest_framework import generics

# Pattern A: Separate views for list/create and detail
class PostList(generics.ListCreateAPIView):
    """
    GET  /posts/  → list()
    POST /posts/  → create()
    
    Inheritance: ListCreateAPIView
      → ListModelMixin + CreateModelMixin + GenericAPIView
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    
    def perform_create(self, serializer):
        """Hook: Called after validation, before save"""
        serializer.save(author=self.request.user)


class PostDetail(generics.RetrieveUpdateDestroyAPIView):
    """
    GET    /posts/{pk}/  → retrieve()
    PUT    /posts/{pk}/  → update()
    PATCH  /posts/{pk}/  → partial_update()
    DELETE /posts/{pk}/  → destroy()
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]


# Pattern B: Granular control with specific mixins
from rest_framework import mixins, generics

class PostList(mixins.ListModelMixin,
               mixins.CreateModelMixin,
               generics.GenericAPIView):
    """
    Same as ListCreateAPIView but explicit about mixins
    Use when you need to understand/customize mixin behavior
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    
    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)
    
    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)


# Pattern C: Read-only view
class PostList(generics.ListAPIView):
    """Only GET /posts/ - no create"""
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    filter_backends = [filters.SearchFilter, filters.OrderingFilter]
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'title']
```

**Available Generic Views:**

```python
# Single action views
CreateAPIView          # POST only
ListAPIView            # GET list only
RetrieveAPIView        # GET detail only
DestroyAPIView         # DELETE only
UpdateAPIView          # PUT/PATCH only

# Combined views
ListCreateAPIView              # GET list + POST
RetrieveUpdateAPIView          # GET detail + PUT/PATCH
RetrieveDestroyAPIView         # GET detail + DELETE
RetrieveUpdateDestroyAPIView   # GET detail + PUT/PATCH + DELETE
```

**When to use:**
- **Standard CRUD**: Operations fit generic patterns
- **Less boilerplate**: Than APIView
- **More control**: Than ViewSet
- **No router needed**: Manual URL configuration

**Question:** *Generic views vs ViewSets - which is better?*
**Answer:**
- **Generic Views**: Better for non-standard REST patterns, manual URLs
- **ViewSets**: Better for standard CRUD, automatic URL routing
- **Enterprise**: Mix both based on needs

---

#### Pattern 4: ViewSets (Router-friendly)

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

class PostViewSet(viewsets.ModelViewSet):
    """
    Complete CRUD operations with router support
    
    Provides:
    - list()    → GET  /posts/
    - create()  → POST /posts/
    - retrieve() → GET  /posts/{pk}/
    - update()  → PUT  /posts/{pk}/
    - partial_update() → PATCH /posts/{pk}/
    - destroy() → DELETE /posts/{pk}/
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
    filter_backends = [filters.SearchFilter, filters.OrderingFilter]
    search_fields = ['title', 'content']
    
    def get_queryset(self):
        """
        Customize queryset based on action
        Called for every request
        """
        queryset = super().get_queryset()
        
        if self.action == 'list':
            # Optimize list queries
            queryset = queryset.select_related('author').only(
                'id', 'title', 'author__username', 'created_at'
            )
        elif self.action == 'retrieve':
            # Full data for detail view
            queryset = queryset.select_related('author').prefetch_related('comments')
        
        return queryset
    
    def get_serializer_class(self):
        """Different serializers for different actions"""
        if self.action == 'list':
            return PostListSerializer
        elif self.action == 'retrieve':
            return PostDetailSerializer
        return PostSerializer
    
    def perform_create(self, serializer):
        """Hook before saving"""
        serializer.save(author=self.request.user)
    
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """
        Custom action: POST /posts/{pk}/publish/
        
        detail=True  → requires pk in URL
        detail=False → collection-level action
        """
        post = self.get_object()
        post.status = 'published'
        post.published_at = timezone.now()
        post.save()
        
        return Response({'status': 'published'})
    
    @action(detail=False, methods=['get'])
    def trending(self, request):
        """
        Custom action: GET /posts/trending/
        Collection-level action (no pk needed)
        """
        trending_posts = self.get_queryset().filter(
            created_at__gte=timezone.now() - timedelta(days=7)
        ).annotate(
            like_count=Count('likes')
        ).order_by('-like_count')[:10]
        
        serializer = self.get_serializer(trending_posts, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['post'], permission_classes=[permissions.IsAuthenticated])
    def like(self, request, pk=None):
        """
        POST /posts/{pk}/like/
        Custom permissions for specific action
        """
        post = self.get_object()
        Like.objects.get_or_create(user=request.user, post=post)
        return Response({'status': 'liked'})
```

**ViewSet Variations:**

```python
# 1. ModelViewSet - Full CRUD
class PostViewSet(viewsets.ModelViewSet):
    # Includes: list, create, retrieve, update, partial_update, destroy
    pass

# 2. ReadOnlyModelViewSet - Read-only
class PostViewSet(viewsets.ReadOnlyModelViewSet):
    # Includes: list, retrieve only
    pass

# 3. GenericViewSet - Custom combinations
class PostViewSet(viewsets.GenericViewSet):
    # No actions by default, add mixins as needed
    pass

# 4. GenericViewSet with specific mixins
class PostViewSet(mixins.ListModelMixin,
                  mixins.RetrieveModelMixin,
                  viewsets.GenericViewSet):
    # Only list and retrieve, no create/update/delete
    queryset = Post.objects.all()
    serializer_class = PostSerializer
```

**When to use:**
- **Standard REST**: CRUD operations
- **Router-based**: Automatic URL generation
- **Multiple actions**: Custom actions with @action
- **DRY code**: Minimal boilerplate

---

#### Pattern 5: Async Views (Django 4.1+, Real-time)

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from asgiref.sync import sync_to_async
import asyncio

class AsyncPostList(APIView):
    """
    Async view for concurrent operations
    Useful for: External API calls, I/O-bound operations
    """
    
    async def get(self, request):
        """
        Async GET handler
        Can await multiple operations concurrently
        """
        
        # Fetch data from multiple sources concurrently
        posts_task = sync_to_async(list)(
            Post.objects.select_related('author').all()
        )
        
        # External API call (async)
        async with aiohttp.ClientSession() as session:
            analytics_task = session.get('https://analytics.api/posts/stats')
            
            # Run concurrently
            posts, analytics_response = await asyncio.gather(
                posts_task,
                analytics_task
            )
        
        analytics_data = await analytics_response.json()
        
        # Serialize posts
        serializer = PostSerializer(posts, many=True)
        
        return Response({
            'posts': serializer.data,
            'analytics': analytics_data
        })
    
    async def post(self, request):
        """Async POST handler"""
        serializer = PostSerializer(data=request.data)
        
        # Sync operation wrapped in sync_to_async
        is_valid = await sync_to_async(serializer.is_valid)(raise_exception=True)
        
        # Save to database (sync operation)
        post = await sync_to_async(serializer.save)(author=request.user)
        
        # Trigger async tasks
        await asyncio.gather(
            send_notification_async(post.author, 'Post created'),
            update_search_index_async(post.id),
            notify_followers_async(post.id)
        )
        
        return Response(serializer.data, status=201)
```

**When to use:**
- **I/O-bound**: External API calls, file operations
- **Concurrent**: Multiple independent operations
- **Real-time**: WebSocket integration
- **High throughput**: Handle more concurrent requests

**Not for:**
- **CPU-bound**: Heavy computations (use Celery instead)
- **Simple CRUD**: Sync is simpler and sufficient

---

#### Pattern 6: Atomic Views (Transaction Safety)

```python
from django.db import transaction
from rest_framework import viewsets, status
from rest_framework.response import Response
from rest_framework.decorators import action

class OrderViewSet(viewsets.ModelViewSet):
    """
    Enterprise pattern: Ensure data consistency with transactions
    """
    queryset = Order.objects.all()
    serializer_class = OrderSerializer
    
    @transaction.atomic
    def create(self, request):
        """
        All database operations in atomic block
        If ANY operation fails, ALL rollback
        """
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        
        # Multiple related operations
        order = serializer.save(user=request.user)
        
        # Update inventory
        for item in request.data['items']:
            product = Product.objects.select_for_update().get(id=item['product_id'])
            
            if product.stock < item['quantity']:
                # This raise will rollback the entire transaction
                raise ValidationError(f"Insufficient stock for {product.name}")
            
            product.stock -= item['quantity']
            product.save()
        
        # Create payment record
        Payment.objects.create(
            order=order,
            amount=order.total,
            status='pending'
        )
        
        # Update user stats
        request.user.total_orders += 1
        request.user.save()
        
        # If we reach here, commit all changes
        return Response(serializer.data, status=status.HTTP_201_CREATED)
    
    @action(detail=True, methods=['post'])
    def cancel(self, request, pk=None):
        """Cancel order with automatic rollback"""
        with transaction.atomic():
            order = self.get_object()
            
            if order.status != 'pending':
                raise ValidationError("Cannot cancel processed order")
            
            # Restore inventory
            for item in order.items.all():
                product = Product.objects.select_for_update().get(id=item.product_id)
                product.stock += item.quantity
                product.save()
            
            # Refund payment
            if hasattr(order, 'payment'):
                order.payment.status = 'refunded'
                order.payment.save()
            
            # Update order status
            order.status = 'cancelled'
            order.save()
            
            # All changes committed together
            return Response({'status': 'cancelled'})
```

**Transaction decorators vs context managers:**

```python
# Method 1: Decorator (cleaner for entire method)
@transaction.atomic
def create(self, request):
    # Everything in this method is atomic
    pass

# Method 2: Context manager (fine-grained control)
def create(self, request):
    # Some non-atomic code here
    
    with transaction.atomic():
        # Only this block is atomic
        pass
    
    # More non-atomic code

# Method 3: Savepoints (nested transactions)
def complex_operation(self, request):
    with transaction.atomic():
        # Outer transaction
        
        try:
            with transaction.atomic():
                # Inner transaction (savepoint)
                risky_operation()
        except Exception:
            # Inner rolled back, outer continues
            pass
        
        safe_operation()
```

---

#### Pattern 7: Cached Views (Performance)

```python
from django.core.cache import cache
from# Django REST Framework: Complete Engineering Guide

## Table of Contents
1. [Request-Response Flow](#request-response-flow)
2. [URL Routing Engineering](#url-routing-engineering)
3. [Views: Execution Patterns](#views-execution-patterns)
4. [Serializers: Data Transformation](#serializers-data-transformation)
5. [Models: Database Layer](#models-database-layer)
6. [Renderers: Response Formatting](#renderers-response-formatting)
7. [First Principles & Design Patterns](#first-principles--design-patterns)
8. [Circle of Competence](#circle-of-competence)
9. [Efficient Code Organization](#efficient-code-organization)

---

## Request-Response Flow

### The Complete Journey: HTTP Request → Response

```
┌─────────────┐
│HTTP Request │  GET /api/posts/1/
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  WSGI Server    │  Gunicorn/uWSGI receives raw HTTP
│  (Gunicorn)     │  Parses headers, body, method
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│Django Middleware│  Process request (auth, CORS, etc.)
│     Stack       │  SecurityMiddleware, SessionMiddleware
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  URL Router     │  urls.py matches pattern
│  (URLconf)      │  Extracts path parameters: pk=1
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  DRF View       │  View.dispatch() → determine HTTP method
│  (ViewSet)      │  Run authentication & permissions
│                 │  Execute retrieve() method
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│   Queryset      │  ORM: Post.objects.get(pk=1)
│   Execution     │  SQL: SELECT * FROM posts WHERE id=1
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Model Layer    │  Post instance created
│  (ORM Object)   │  Python object with attributes
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Serializer     │  PostSerializer(post_instance)
│  (to_representation) Converts model → dict
│                 │  Handles nested relations
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│   Renderer      │  JSONRenderer converts dict → JSON
│  (JSONRenderer) │  Adds Content-Type header
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│DRF Response Obj │  Response(serializer.data)
│                 │  Sets status code, headers
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│Django Middleware│  Process response (reverse order)
│     Stack       │  Add headers, compress, etc.
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  HTTP Response  │  Status: 200 OK
│                 │  Body: {"id":1,"title":"..."}
└─────────────────┘
```

---

## URL Routing Engineering

### How URL Resolution Works

**File: `urls.py`**

```python
# myproject/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from posts.views import PostViewSet

# Router automatically generates URL patterns
router = DefaultRouter()
router.register(r'posts', PostViewSet, basename='post')

urlpatterns = [
    path('api/', include(router.urls)),
]
```

#### Line-by-Line Execution:

1. **`DefaultRouter()`** - Creates router instance
   - Router maintains registry: `{prefix: viewset}` mapping
   - Generates standard RESTful URL patterns

2. **`router.register(r'posts', PostViewSet, basename='post')`**
   - Registers ViewSet with 'posts' prefix
   - Creates URL patterns:
     ```
     /api/posts/          → list(), create()
     /api/posts/{pk}/     → retrieve(), update(), destroy()
     /api/posts/{pk}/custom_action/ → custom methods
     ```

3. **`include(router.urls)`** - Merges router URLs into main URLconf

### URL Resolution Process:

```python
# When request arrives: GET /api/posts/1/

# Step 1: Django matches 'api/' in main urls.py
# Step 2: Router matches 'posts/1/' pattern
# Step 3: Router extracts pk=1
# Step 4: Router determines method (GET) → calls retrieve()
# Step 5: View receives: retrieve(request, pk='1')
```

### Alternative URL Patterns (Manual):

```python
# Without Router - Manual URL definition
from posts.views import PostListCreateView, PostDetailView

urlpatterns = [
    path('api/posts/', PostListCreateView.as_view(), name='post-list'),
    path('api/posts/<int:pk>/', PostDetailView.as_view(), name='post-detail'),
]
```

**Differences:**
- **Router**: Auto-generates URLs, handles ViewSets, less boilerplate
- **Manual**: More control, works with APIView/GenericView, explicit

---

## Views: Execution Patterns

### Three Ways to Write Views

#### 1. Function-Based View (FBV)

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view(['GET', 'POST'])
def post_list(request):
    """
    Execution Flow:
    1. Decorator wraps function
    2. Converts Django request → DRF Request
    3. Executes function body
    4. Converts return value → DRF Response
    """
    if request.method == 'GET':
        posts = Post.objects.all()
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = PostSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=201)
        return Response(serializer.errors, status=400)
```

**Execution Path:**
```
@api_view decorator
  → wraps function
  → request arrives
  → check HTTP method against allowed methods
  → execute function
  → wrap return in Response object
```

**Pros:** Simple, explicit, easy to understand
**Cons:** More code duplication, less DRY

---

#### 2. Class-Based View (APIView)

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class PostList(APIView):
    """
    Execution Flow:
    1. dispatch() receives request
    2. Runs authentication_classes
    3. Runs permission_classes
    4. Calls appropriate method (get/post/put/delete)
    5. Returns Response
    """
    
    authentication_classes = [TokenAuthentication]
    permission_classes = [IsAuthenticated]
    
    def get(self, request, format=None):
        """
        When GET request arrives:
        1. dispatch() → self.initial(request) [auth/permissions]
        2. dispatch() → self.get(request)
        3. Returns Response object
        """
        posts = Post.objects.all()
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)
    
    def post(self, request, format=None):
        serializer = PostSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

**Execution Sequence:**
```python
# Internal flow for: POST /api/posts/

APIView.as_view()  # Returns view function
  ↓
dispatch(request, *args, **kwargs)
  ↓
self.initial(request)  # Authentication & Permissions
  ↓
  ├─ self.perform_authentication(request)
  │    → runs TokenAuthentication.authenticate()
  │    → sets request.user
  ↓
  ├─ self.check_permissions(request)
  │    → IsAuthenticated.has_permission()
  │    → raises PermissionDenied if False
  ↓
  └─ self.check_throttles(request)
       → rate limiting
  ↓
handler = getattr(self, request.method.lower())  # Gets self.post
  ↓
response = handler(request, *args, **kwargs)  # Executes self.post()
  ↓
self.response = self.finalize_response(request, response, *args, **kwargs)
  ↓
return response
```

**Pros:** Class structure, reusable, middleware hooks
**Cons:** More boilerplate than ViewSets

---

#### 3. Generic Views & ViewSets

```python
from rest_framework import viewsets, mixins
from rest_framework.decorators import action

class PostViewSet(viewsets.ModelViewSet):
    """
    ModelViewSet = GenericViewSet + all CRUD mixins
    
    Inheritance chain:
    ModelViewSet
      ↳ mixins.CreateModelMixin
      ↳ mixins.RetrieveModelMixin
      ↳ mixins.UpdateModelMixin
      ↳ mixins.DestroyModelMixin
      ↳ mixins.ListModelMixin
      ↳ GenericViewSet
          ↳ ViewSetMixin
          ↳ GenericAPIView
              ↳ APIView
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def get_queryset(self):
        """
        Called by get_object() and list()
        Execution: Every time queryset is accessed
        """
        queryset = super().get_queryset()
        
        # Optimize based on action
        if self.action == 'list':
            queryset = queryset.select_related('author')
        
        return queryset
    
    def perform_create(self, serializer):
        """
        Called by create() after validation
        Execution: POST request, after is_valid()
        """
        serializer.save(author=self.request.user)
    
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """
        Custom action: POST /api/posts/{pk}/publish/
        
        Execution flow:
        1. Router recognizes 'publish' action
        2. dispatch() routes to this method
        3. Method executes custom logic
        """
        post = self.get_object()  # Calls get_queryset(), checks permissions
        post.status = 'published'
        post.save()
        return Response({'status': 'published'})
```

**ViewSet Method Execution Order:**

```python
# For: GET /api/posts/1/

1. PostViewSet.as_view({'get': 'retrieve'})
   ↓ Router maps HTTP method → ViewSet method
   
2. dispatch(request, pk=1)
   ↓
   
3. initial(request)
   ↓ Run authentication & permissions
   
4. retrieve(request, pk=1)  # From RetrieveModelMixin
   ↓
   
5. get_object()
   ├─ get_queryset()  # Returns Post.objects.all()
   ├─ filter_queryset(queryset)  # Apply filters
   ├─ queryset.get(pk=1)  # Database query executes HERE
   └─ check_object_permissions(request, obj)
   ↓
   
6. get_serializer(instance)
   ↓
   
7. return Response(serializer.data)
```

**Comparison Table:**

| Feature | FBV | APIView | ViewSet |
|---------|-----|---------|---------|
| Lines of code | Most | Medium | Least |
| Flexibility | High | High | Medium |
| Reusability | Low | Medium | High |
| Router support | No | No | Yes |
| Auto URL generation | No | No | Yes |
| Learning curve | Easiest | Medium | Steepest |
| Best for | Simple endpoints | Custom logic | CRUD operations |

---

## Serializers: Data Transformation

### How Serialization Works

```python
from rest_framework import serializers

class PostSerializer(serializers.ModelSerializer):
    """
    Serializer has two main phases:
    1. Deserialization: JSON/Form data → Python objects → Model instances
    2. Serialization: Model instances → Python dict → JSON
    """
    
    author_name = serializers.CharField(source='author.username', read_only=True)
    comments_count = serializers.IntegerField(source='comments.count', read_only=True)
    url = serializers.HyperlinkedIdentityField(view_name='post-detail')
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author_name', 'comments_count', 'url']
        read_only_fields = ['id', 'created_at']
    
    def validate_title(self, value):
        """
        Field-level validation
        Execution: During is_valid() call, for 'title' field
        """
        if len(value) < 5:
            raise serializers.ValidationError("Title too short")
        return value
    
    def validate(self, data):
        """
        Object-level validation
        Execution: After all field validations pass
        """
        if 'title' in data and 'spam' in data['title'].lower():
            raise serializers.ValidationError("Spam detected")
        return data
    
    def create(self, validated_data):
        """
        Execution: When serializer.save() is called (no instance provided)
        """
        return Post.objects.create(**validated_data)
    
    def update(self, instance, validated_data):
        """
        Execution: When serializer.save() is called (instance provided)
        """
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.save()
        return instance
```

### Serialization Flow (Model → JSON):

```python
# Code: serializer = PostSerializer(post_instance)

Step 1: __init__(instance=post_instance)
  ↓ Stores instance
  ↓ Initializes fields
  
Step 2: Access .data property
  ↓
  
Step 3: to_representation(instance)  # Main conversion method
  ↓
  
  For each field in fields:
    ├─ field.get_attribute(instance)  # Gets model attribute
    │   Example: author_name → instance.author.username
    ↓
    ├─ field.to_representation(value)  # Converts to primitive
    │   CharField → str
    │   IntegerField → int
    │   DateTimeField → ISO format string
    ↓
    └─ result[field_name] = primitive_value
  
Step 4: Returns OrderedDict
  {
    'id': 1,
    'title': 'My Post',
    'author_name': 'john',
    'comments_count': 5
  }
```

### Deserialization Flow (JSON → Model):

```python
# Code: serializer = PostSerializer(data=request.data)
#       serializer.is_valid(raise_exception=True)
#       serializer.save()

Step 1: __init__(data={'title': 'New Post', 'content': '...'})
  ↓
  
Step 2: is_valid(raise_exception=True)
  ↓
  ├─ Run field validators
  │   ├─ validate_title('New Post')  # Field-level
  │   ├─ validate_content('...')
  │   └─ Store in validated_data
  ↓
  ├─ Run object validator
  │   └─ validate(validated_data)  # Object-level
  ↓
  └─ If any fail → raise ValidationError
  
Step 3: save()
  ↓
  ├─ Check if instance exists
  │   ├─ If NO → create(validated_data)
  │   └─ If YES → update(instance, validated_data)
  ↓
  └─ Return model instance
```

### Nested Serializers Execution:

```python
class CommentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Comment
        fields = ['id', 'text', 'author']

class PostSerializer(serializers.ModelSerializer):
    comments = CommentSerializer(many=True, read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'comments']
```

**Execution for nested serialization:**

```python
# post.comments.all() → [comment1, comment2, comment3]

PostSerializer(post).data
  ↓
  to_representation(post)
    ├─ title → post.title  # Simple field
    ↓
    ├─ comments → CommentSerializer(post.comments.all(), many=True)
    │   ↓
    │   to_representation([comment1, comment2, comment3])
    │     ├─ For comment1:
    │     │   └─ CommentSerializer(comment1).data
    │     ├─ For comment2:
    │     │   └─ CommentSerializer(comment2).data
    │     └─ For comment3:
    │         └─ CommentSerializer(comment3).data
    │   ↓
    │   Returns: [
    │     {'id': 1, 'text': '...', 'author': 'john'},
    │     {'id': 2, 'text': '...', 'author': 'jane'},
    │     {'id': 3, 'text': '...', 'author': 'bob'}
    │   ]
    └─ Result includes nested data
```

---

## Models: Database Layer

### ORM Query Execution

```python
from django.db import models

class Post(models.Model):
    """
    Model defines:
    1. Database table structure
    2. Field types and constraints
    3. Relationships
    4. Business logic methods
    """
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey('User', on_delete=models.CASCADE, related_name='posts')
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        db_table = 'posts'
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['created_at']),
            models.Index(fields=['author', 'created_at']),
        ]
    
    def __str__(self):
        return self.title
```

### Query Execution Path:

```python
# ViewSet: post = Post.objects.get(pk=1)

Step 1: Post.objects → Returns Manager instance
  ↓
  
Step 2: Manager.get(pk=1)
  ↓
  
Step 3: QuerySet created (lazy)
  QuerySet: SELECT * FROM posts WHERE id = 1
  ↓ NOT executed yet!
  
Step 4: QuerySet.get() forces evaluation
  ↓
  
Step 5: Django's SQL compiler
  ├─ Builds SQL query
  │   SELECT "posts"."id", "posts"."title", "posts"."content",
  │          "posts"."author_id", "posts"."created_at"
  │   FROM "posts"
  │   WHERE "posts"."id" = 1
  ↓
  
Step 6: Database executes query
  ↓
  
Step 7: Django creates Post instance
  post = Post(
    id=1,
    title='...',
    content='...',
    author_id=5,
    created_at=datetime(...)
  )
  ↓
  
Step 8: Returns Post object
```

### N+1 Query Problem:

```python
# BAD: Causes N+1 queries
posts = Post.objects.all()  # 1 query
for post in posts:
    print(post.author.username)  # N queries (one per post)

# Execution:
# Query 1: SELECT * FROM posts
# Query 2: SELECT * FROM users WHERE id = post1.author_id
# Query 3: SELECT * FROM users WHERE id = post2.author_id
# ... N more queries


# GOOD: Use select_related (for ForeignKey)
posts = Post.objects.select_related('author').all()  # 1 query with JOIN
for post in posts:
    print(post.author.username)  # No additional queries

# Execution:
# Query 1: SELECT * FROM posts INNER JOIN users ON posts.author_id = users.id


# GOOD: Use prefetch_related (for ManyToMany / Reverse FK)
posts = Post.objects.prefetch_related('comments').all()  # 2 queries

# Execution:
# Query 1: SELECT * FROM posts
# Query 2: SELECT * FROM comments WHERE post_id IN (1, 2, 3, ...)
# Django joins in Python
```

---

## Renderers: Response Formatting

### How Rendering Works

```python
from rest_framework.renderers import JSONRenderer, BrowsableAPIRenderer

class PostViewSet(viewsets.ModelViewSet):
    renderer_classes = [JSONRenderer, BrowsableAPIRenderer]
```

**Rendering Flow:**

```python
# View returns: Response({'id': 1, 'title': 'Post'})

Step 1: Response object created
  Response(
    data={'id': 1, 'title': 'Post'},
    status=200,
    headers={}
  )
  ↓
  
Step 2: finalize_response(request, response)
  ↓
  
Step 3: Perform content negotiation
  ├─ Check Accept header: Accept: application/json
  ├─ Match with available renderers
  └─ Select JSONRenderer
  ↓
  
Step 4: JSONRenderer.render(data, media_type, renderer_context)
  ├─ Convert OrderedDict to JSON string
  │   json.dumps({'id': 1, 'title': 'Post'})
  ↓
  └─ Returns: b'{"id":1,"title":"Post"}'
  
Step 5: Set response headers
  ├─ Content-Type: application/json
  └─ Content-Length: 28
  ↓
  
Step 6: Return HTTP response
  HTTP/1.1 200 OK
  Content-Type: application/json
  Content-Length: 28
  
  {"id":1,"title":"Post"}
```

### Custom Renderer:

```python
from rest_framework.renderers import BaseRenderer

class CSVRenderer(BaseRenderer):
    media_type = 'text/csv'
    format = 'csv'
    
    def render(self, data, media_type=None, renderer_context=None):
        """
        Execution: When Accept: text/csv header present
        """
        if not isinstance(data, list):
            data = [data]
        
        # Convert to CSV format
        csv_data = []
        if data:
            headers = ','.join(data[0].keys())
            csv_data.append(headers)
            
            for item in data:
                row = ','.join(str(v) for v in item.values())
                csv_data.append(row)
        
        return '\n'.join(csv_data).encode('utf-8')
```

---

## First Principles & Design Patterns

### First Principles Thinking

**Question: "Why does DRF need Serializers?"**

**First Principle Reasoning:**

1. **Foundation**: HTTP can only transmit text (bytes)
2. **Problem**: Python objects can't be sent over HTTP
3. **Core Need**: Conversion between Python ↔ Text
4. **Additional Needs**: 
   - Validation (data integrity)
   - Different representations (JSON, XML, CSV)
   - Security (field-level permissions)

**Result**: Serializer = Converter + Validator + Presenter

### Design Patterns in DRF

#### 1. **Mixin Pattern**

```python
# Instead of inheritance hell, compose behaviors

class ListModelMixin:
    def list(self, request, *args, **kwargs):
        queryset = self.filter_queryset(self.get_queryset())
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)

class CreateModelMixin:
    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)
        return Response(serializer.data, status=201)

# Compose what you need
class PostViewSet(ListModelMixin, CreateModelMixin, GenericViewSet):
    pass  # Gets list() and create() methods
```

**Why Mixins?**
- **Principle**: Composition over inheritance
- **Benefit**: Pick exactly what you need
- **Flexibility**: Easy to override specific methods

#### 2. **Adapter Pattern**

```python
# DRF adapts Django's request/response to REST semantics

# Django HttpRequest → DRF Request (adapter)
class Request:
    def __init__(self, request):
        self._request = request  # Wraps Django request
        self._data = None
        
    @property
    def data(self):
        """Adapter: Unifies POST, PUT, PATCH data"""
        if self._data is None:
            self._data = self._parse()
        return self._data
```

#### 3. **Strategy Pattern**

```python
# Authentication strategies

class TokenAuthentication(BaseAuthentication):
    def authenticate(self, request):
        token = request.META.get('HTTP_AUTHORIZATION')
        # Strategy: Token-based auth
        return (user, token)

class SessionAuthentication(BaseAuthentication):
    def authenticate(self, request):
        # Strategy: Session-based auth
        return (request.user, None)

# View chooses strategy
class PostViewSet(viewsets.ModelViewSet):
    authentication_classes = [TokenAuthentication]  # Strategy selection
```

---

## Circle of Competence

### Level 1: Beginner (Focus Area)
**Master these first:**

```python
# 1. Simple serializers
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content']

# 2. Basic ViewSets
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

# 3. Simple URL routing
router.register(r'posts', PostViewSet)
```

**Why start here?**
- Covers 80% of use cases
- Builds foundation for advanced concepts
- Quick wins, immediate productivity

### Level 2: Intermediate (Expand Gradually)

```python
# Custom actions, filtering, permissions
class PostViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticatedOrReadOnly]
    filter_backends = [SearchFilter]
    search_fields = ['title']
    
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        # Custom business logic
        pass
```

### Level 3: Advanced (Master Later)

```python
# Custom renderers, complex nested serializers, optimization
class OptimizedPostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return Post.objects.select_related('author').prefetch_related(
            Prefetch('comments', queryset=Comment.objects.select_related('user'))
        )
```

**Progression Strategy:**
1. Master Level 1 thoroughly (2-3 months)
2. Identify real problems that need Level 2 solutions
3. Only explore Level 3 when you have specific performance needs

---

## Efficient Code Organization

### Project Structure

```
myproject/
├── myproject/
│   ├── settings.py
│   ├── urls.py              # Main URL router
│   └── wsgi.py
│
├── apps/
│   ├── posts/
│   │   ├── models.py        # Database models
│   │   ├── serializers.py   # Serializer classes
│   │   ├── views.py         # ViewSets and Views
│   │   ├── urls.py          # App-specific URLs
│   │   ├── permissions.py   # Custom permissions
│   │   ├── filters.py       # Custom filters
│   │   └── tests.py         # Unit tests
│   │
│   └── users/
│       └── ...
│
├── core/
│   ├── permissions.py       # Shared permissions
│   ├── pagination.py        # Custom pagination
│   └── renderers.py         # Custom renderers
│
└── manage.py
```

### When to Use Each Pattern

#### Use **Function-Based Views** when:
```python
# Simple, one-off endpoints
# No reuse needed
# Quick prototypes

@api_view(['POST'])
def trigger_export(request):
    # One-time utility endpoint
    export_data.delay()
    return Response({'status': 'started'})
```

#### Use **APIView** when:
```python
# Custom logic not fitting CRUD
# Multiple related operations
# Need class structure but not CRUD

class StatsView(APIView):
    def get(self, request):
        # Complex aggregation logic
        stats = calculate_statistics()
        return Response(stats)
```

#### Use **GenericAPIView + Mixins** when:
```python
# Need some CRUD, but not all
# Custom combinations

class PostListCreate(mixins.ListModelMixin,
                     mixins.CreateModelMixin,
                     generics.GenericAPIView):
    # Only list and create, no detail views
    pass
```

#### Use **ViewSets** when:
```python
# Full CRUD operations
# Standard RESTful resource
# Want router auto-generation

class PostViewSet(viewsets.ModelViewSet):
    # Complete CRUD + custom actions
    queryset = Post.objects.all()
    serializer_class = PostSerializer
```

### Efficiency Rules

1. **Don't Optimize Prematurely**
   ```python
   # Start simple
   queryset = Post.objects.all()
   
   # Optimize when you measure a problem
   queryset = Post.objects.select_related('author').prefetch_related('tags')
   ```

2. **Measure First, Then Optimize**
   ```python
   from django.db import connection
   from django.test.utils import override_settings
   
   @override_settings(DEBUG=True)
   def test_query_count():
       Post.objects.all()[0].author.username
       print(len(connection.queries))  # Measure queries
   ```

3. **Use the Simplest Solution**
   ```python
   # Too complex for simple task
   class PostViewSet(viewsets.ModelViewSet):
       # ... 50 lines of code
   
   # Better for simple need
   @api_view(['GET'])
   def get_posts(request):
       posts = Post.objects.all()
       return Response(PostSerializer(posts, many=True).data)
   ```

---

## Deep Dive: Request-Response Lifecycle

### Complete Execution Trace

Let's trace a real request: `POST /api/posts/` with authentication.

```python
# Request payload:
POST /api/posts/ HTTP/1.1
Host: api.example.com
Authorization: Token abc123xyz
Content-Type: application/json

{
  "title": "My New Post",
  "content": "This is the content"
}
```

#### Execution Timeline:

```python
# ==================== PHASE 1: SERVER LAYER ====================
# Time: 0ms

# 1. Gunicorn/uWSGI receives raw socket data
worker.handle_request(socket_data)

# 2. Parse HTTP protocol
http_parser.parse(socket_data)
  → method: POST
  → path: /api/posts/
  → headers: {'Authorization': 'Token abc123xyz', 'Content-Type': 'application/json'}
  → body: '{"title":"My New Post","content":"This is the content"}'


# ==================== PHASE 2: DJANGO MIDDLEWARE ====================
# Time: 2ms

# 3. Django creates HttpRequest object
request = HttpRequest()
request.method = 'POST'
request.path = '/api/posts/'
request.META['HTTP_AUTHORIZATION'] = 'Token abc123xyz'
request.body = b'{"title":"My New Post","content":"This is the content"}'

# 4. Middleware stack (top to bottom)
SecurityMiddleware(request)          # HTTPS redirect, security headers
SessionMiddleware(request)           # Load session data
CommonMiddleware(request)            # URL handling, ETags
CsrfViewMiddleware(request)          # CSRF protection (skipped for API)
AuthenticationMiddleware(request)    # Sets request.user (basic Django auth)
MessageMiddleware(request)           # Flash messages
XFrameOptionsMiddleware(request)     # Clickjacking protection


# ==================== PHASE 3: URL RESOLUTION ====================
# Time: 4ms

# 5. URLconf resolution
urlpatterns = [
    path('api/', include(router.urls)),  # Match found!
]

# 6. Router pattern matching
router.urls = [
    path('posts/', PostViewSet.as_view({'get': 'list', 'post': 'create'})),
    path('posts/<pk>/', PostViewSet.as_view({'get': 'retrieve', ...})),
]

# Match: 'posts/' → PostViewSet
# Method: POST → action='create'

# 7. View instantiation
view_func = PostViewSet.as_view({'post': 'create'})
# This returns a callable that wraps the ViewSet


# ==================== PHASE 4: DRF REQUEST WRAPPER ====================
# Time: 6ms

# 8. DRF wraps Django request
from rest_framework.request import Request

drf_request = Request(
    request=django_request,
    parsers=[JSONParser(), FormParser(), MultiPartParser()],
    authenticators=[TokenAuthentication()],
    negotiator=DefaultContentNegotiation(),
    parser_context={'view': view, 'args': (), 'kwargs': {}}
)

# 9. Parse request data (lazy - happens on first access to request.data)
drf_request.data  # Triggers parsing
  ↓
parser = JSONParser()
parsed_data = parser.parse(
    stream=request.body,
    media_type='application/json',
    parser_context={}
)
# Result: {'title': 'My New Post', 'content': 'This is the content'}


# ==================== PHASE 5: VIEW DISPATCH ====================
# Time: 8ms

# 10. ViewSet.dispatch() called
PostViewSet.dispatch(request=drf_request)

# 11. Initialize the view
self.action = 'create'
self.format_kwarg = None
self.args = ()
self.kwargs = {}

# 12. initial() - Pre-processing
self.initial(request)
  ↓
  # 12a. Authentication
  self.perform_authentication(request)
    ↓
    authenticator = TokenAuthentication()
    user, token = authenticator.authenticate(request)
      ↓
      # Extract token from header
      auth_header = request.META.get('HTTP_AUTHORIZATION')  # 'Token abc123xyz'
      token_key = auth_header.split(' ')[1]  # 'abc123xyz'
      
      # Database query
      token_obj = Token.objects.select_related('user').get(key='abc123xyz')
      # SQL: SELECT * FROM authtoken_token 
      #      INNER JOIN auth_user ON authtoken_token.user_id = auth_user.id
      #      WHERE authtoken_token.key = 'abc123xyz'
      
      user = token_obj.user
      request.user = user  # Set authenticated user
      request.auth = token_obj
  
  # 12b. Permission check
  self.check_permissions(request)
    ↓
    for permission in self.permission_classes:
        perm = IsAuthenticatedOrReadOnly()
        if not perm.has_permission(request, self):
            raise PermissionDenied()
    # User is authenticated → Permission granted
  
  # 12c. Throttling
  self.check_throttles(request)
    ↓
    throttles = [UserRateThrottle()]
    for throttle in throttles:
        if not throttle.allow_request(request, self):
            raise Throttled()
    # Check rate: user made 50 requests today, limit is 1000 → Allowed


# ==================== PHASE 6: ACTION EXECUTION ====================
# Time: 12ms

# 13. Dispatch to action handler
handler = getattr(self, 'create')  # Get create() method
response = handler(request)

# 14. CreateModelMixin.create() executes
def create(self, request, *args, **kwargs):
    # 14a. Get serializer
    serializer = self.get_serializer(data=request.data)
    # PostSerializer(data={'title': 'My New Post', 'content': '...'})
    
    # 14b. Validation
    serializer.is_valid(raise_exception=True)


# ==================== PHASE 7: SERIALIZER VALIDATION ====================
# Time: 15ms

# 15. Serializer.is_valid() execution
PostSerializer.is_valid(raise_exception=True)
  ↓
  # 15a. Field validation
  self._validated_data = {}
  self._errors = {}
  
  for field_name, field in self.fields.items():
      # Validate 'title' field
      if field_name == 'title':
          primitive_value = self.initial_data.get('title')  # 'My New Post'
          
          # Run field validators
          field.run_validation(primitive_value)
            ↓
            # Check required
            if field.required and primitive_value is None:
                raise ValidationError('This field is required')
            
            # Check max_length
            if len(primitive_value) > field.max_length:
                raise ValidationError('Ensure this field has no more than...')
            
            # Custom field validator
            value = self.validate_title(primitive_value)
              ↓
              if len(value) < 5:
                  raise ValidationError("Title too short")
              return value
          
          self._validated_data['title'] = value
      
      # Same for 'content' field
      if field_name == 'content':
          # ... validation process ...
          self._validated_data['content'] = validated_value
  
  # 15b. Object-level validation
  self._validated_data = self.validate(self._validated_data)
    ↓
    if 'spam' in data.get('title', '').lower():
        raise ValidationError('Spam detected')
    return data
  
  # 15c. Check for errors
  if self._errors:
      raise ValidationError(self._errors)
  
  # Validation passed!
  self.validated_data = {'title': 'My New Post', 'content': 'This is the content'}


# ==================== PHASE 8: DATABASE OPERATION ====================
# Time: 18ms

# 16. Save to database
serializer.save()
  ↓
  # No instance provided → calls create()
  self.perform_create(serializer)
    ↓
    serializer.save(author=request.user)
      ↓
      validated_data['author'] = request.user
      instance = self.create(validated_data)
        ↓
        # ModelSerializer.create()
        post = Post.objects.create(**validated_data)
          ↓
          # Django ORM
          post = Post(
              title='My New Post',
              content='This is the content',
              author=request.user
          )
          post.save()
            ↓
            # SQL generation
            cursor.execute(
                """
                INSERT INTO posts (title, content, author_id, created_at)
                VALUES (%s, %s, %s, %s)
                RETURNING id
                """,
                ['My New Post', 'This is the content', 5, datetime.now()]
            )
            
            # Database executes INSERT
            # Returns: id=42
            
            post.id = 42
      
      return post  # Post instance with id=42


# ==================== PHASE 9: SERIALIZATION (Model → Dict) ====================
# Time: 22ms

# 17. Return Response with serialized data
return Response(serializer.data, status=status.HTTP_201_CREATED)
  ↓
  # Access serializer.data triggers to_representation()
  serializer.data
    ↓
    self.to_representation(instance=post)
      ↓
      ret = OrderedDict()
      
      for field_name, field in self.fields.items():
          # Get attribute from model
          attribute = field.get_attribute(instance)
          
          if field_name == 'id':
              attribute = instance.id  # 42
              ret['id'] = field.to_representation(attribute)  # 42
          
          if field_name == 'title':
              attribute = instance.title  # 'My New Post'
              ret['title'] = field.to_representation(attribute)  # 'My New Post'
          
          if field_name == 'content':
              attribute = instance.content
              ret['content'] = field.to_representation(attribute)
          
          if field_name == 'created_at':
              attribute = instance.created_at  # datetime object
              ret['created_at'] = field.to_representation(attribute)
              # Converts to ISO format: '2025-10-06T10:30:00Z'
          
          if field_name == 'author':
              attribute = instance.author  # User object
              # If author is serialized as nested object
              ret['author'] = field.to_representation(attribute)
              # Returns: {'id': 5, 'username': 'john'}
      
      return ret
      # Result:
      # OrderedDict([
      #     ('id', 42),
      #     ('title', 'My New Post'),
      #     ('content', 'This is the content'),
      #     ('created_at', '2025-10-06T10:30:00Z'),
      #     ('author', {'id': 5, 'username': 'john'})
      # ])


# ==================== PHASE 10: RESPONSE CREATION ====================
# Time: 24ms

# 18. Create Response object
response = Response(
    data=OrderedDict([...]),
    status=201,
    headers={'Location': '/api/posts/42/'}
)


# ==================== PHASE 11: CONTENT NEGOTIATION ====================
# Time: 25ms

# 19. finalize_response()
self.finalize_response(request, response)
  ↓
  # 19a. Content negotiation
  negotiator = DefaultContentNegotiation()
  renderer, media_type = negotiator.select_renderer(
      request=request,
      renderers=self.renderer_classes,
      format_suffix=None
  )
  
  # Check Accept header
  accept_header = request.META.get('HTTP_ACCEPT')  # 'application/json'
  
  # Match with available renderers
  for renderer_class in [JSONRenderer, BrowsableAPIRenderer]:
      if renderer_class.media_type == 'application/json':
          selected_renderer = JSONRenderer()
          break
  
  response.accepted_renderer = selected_renderer
  response.accepted_media_type = 'application/json'


# ==================== PHASE 12: RENDERING ====================
# Time: 26ms

# 20. Render response
rendered_content = response.rendered_content
  ↓
  renderer = JSONRenderer()
  content = renderer.render(
      data=response.data,
      accepted_media_type='application/json',
      renderer_context={
          'view': self,
          'request': request,
          'response': response
      }
  )
  ↓
  # JSONRenderer.render()
  import json
  
  # Handle special types (datetime, Decimal, etc.)
  json_string = json.dumps(
      response.data,
      cls=DjangoJSONEncoder,
      ensure_ascii=False
  )
  
  encoded = json_string.encode('utf-8')
  # Returns: b'{"id":42,"title":"My New Post",...}'


# ==================== PHASE 13: HTTP RESPONSE BUILDING ====================
# Time: 28ms

# 21. Build HTTP response
http_response = HttpResponse(
    content=rendered_content,
    status=201,
    content_type='application/json'
)

# 22. Add headers
http_response['Content-Type'] = 'application/json'
http_response['Content-Length'] = len(rendered_content)
http_response['Location'] = '/api/posts/42/'
http_response['Vary'] = 'Accept'
http_response['Allow'] = 'GET, POST, HEAD, OPTIONS'


# ==================== PHASE 14: MIDDLEWARE RESPONSE ====================
# Time: 29ms

# 23. Response middleware (bottom to top)
XFrameOptionsMiddleware(request, response)   # Add X-Frame-Options
MessageMiddleware(request, response)         # Process messages
AuthenticationMiddleware(request, response)  # Nothing to do
CsrfViewMiddleware(request, response)        # Nothing to do
CommonMiddleware(request, response)          # Add ETag
SessionMiddleware(request, response)         # Save session
SecurityMiddleware(request, response)        # Add security headers


# ==================== PHASE 15: SEND RESPONSE ====================
# Time: 30ms

# 24. Serialize HTTP response
http_response_bytes = b"""HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 156
Location: /api/posts/42/
Vary: Accept
Allow: GET, POST, HEAD, OPTIONS
X-Frame-Options: DENY
X-Content-Type-Options: nosniff

{"id":42,"title":"My New Post","content":"This is the content","created_at":"2025-10-06T10:30:00Z","author":{"id":5,"username":"john"}}"""

# 25. Send to client
socket.send(http_response_bytes)

# ==================== COMPLETE ====================
# Total Time: 30ms
```

### Performance Breakdown

```
Phase                          Time    Percentage
─────────────────────────────────────────────────
Server/Middleware              4ms     13%
URL Resolution                 2ms     7%
DRF Request Wrapping           2ms     7%
Authentication                 4ms     13%
Validation                     3ms     10%
Database INSERT                6ms     20%
Serialization                  4ms     13%
Rendering                      2ms     7%
Response Building              3ms     10%
─────────────────────────────────────────────────
Total                          30ms    100%
```

**Optimization Opportunities:**

1. **Database (20%)** - Biggest impact
   - Use database indexes
   - Optimize queries
   - Connection pooling

2. **Authentication (13%)** - Cache tokens
   - Redis for token storage
   - Reduce DB lookups

3. **Serialization (13%)** - Only serialize needed fields
   - Use `fields` parameter
   - Avoid over-fetching

---

## Advanced Patterns & Trade-offs

### Pattern 1: Flat vs Nested Serializers

#### Flat Structure (Recommended for APIs)
```python
class PostSerializer(serializers.ModelSerializer):
    author_id = serializers.IntegerField()
    author_name = serializers.CharField(source='author.username', read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author_id', 'author_name']

# Output:
{
    "id": 1,
    "title": "My Post",
    "content": "...",
    "author_id": 5,
    "author_name": "john"
}
```

**Pros:**
- Easier to consume for clients
- Clear data structure
- Better for caching
- Simpler validation

**Cons:**
- Data duplication if used in multiple places
- May require multiple requests for related data

#### Nested Structure
```python
class AuthorSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email']

class PostSerializer(serializers.ModelSerializer):
    author = AuthorSerializer(read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author']

# Output:
{
    "id": 1,
    "title": "My Post",
    "content": "...",
    "author": {
        "id": 5,
        "username": "john",
        "email": "john@example.com"
    }
}
```

**Pros:**
- Complete data in one request
- Intuitive structure
- Good for read-heavy operations

**Cons:**
- Complex write operations
- Harder to cache
- Can lead to over-fetching
- Performance issues with deep nesting

**Decision Framework:**
```python
# Use FLAT when:
# - Building public APIs
# - Need high performance
# - Clients vary in needs
# - Write operations are common

# Use NESTED when:
# - Internal microservices
# - Read-only or mostly read
# - Tight coupling with clients
# - UI needs complete data
```

### Pattern 2: Multiple Serializers per ViewSet

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    
    def get_serializer_class(self):
        """
        Dynamic serializer selection based on action
        
        Execution: Called before serializer instantiation
        """
        if self.action == 'list':
            # List: Minimal data, optimized for performance
            return PostListSerializer
        elif self.action == 'retrieve':
            # Detail: Complete data with relationships
            return PostDetailSerializer
        elif self.action == 'create':
            # Create: Writable fields only
            return PostCreateSerializer
        elif self.action == 'update':
            # Update: Different validation rules
            return PostUpdateSerializer
        return PostSerializer  # Default
    
    def get_queryset(self):
        """
        Dynamic queryset optimization based on action
        
        Execution: Every time queryset is accessed
        """
        queryset = super().get_queryset()
        
        if self.action == 'list':
            # List: Only needed fields, no relations
            queryset = queryset.only('id', 'title', 'created_at')
        elif self.action == 'retrieve':
            # Detail: Optimize with select_related
            queryset = queryset.select_related('author').prefetch_related('comments')
        
        return queryset


# Serializers
class PostListSerializer(serializers.ModelSerializer):
    """Minimal data for list view"""
    class Meta:
        model = Post
        fields = ['id', 'title', 'created_at']

class PostDetailSerializer(serializers.ModelSerializer):
    """Complete data for detail view"""
    author = UserSerializer(read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'comments', 'created_at']

class PostCreateSerializer(serializers.ModelSerializer):
    """Writable fields for creation"""
    class Meta:
        model = Post
        fields = ['title', 'content']
    
    def create(self, validated_data):
        # Auto-assign author from request
        validated_data['author'] = self.context['request'].user
        return super().create(validated_data)
```

**Why this pattern?**
- **Principle**: Single Responsibility - each serializer does one thing well
- **Benefit**: Optimized for specific use case
- **Trade-off**: More code, but better performance

### Pattern 3: Serializer Context Usage

```python
class PostSerializer(serializers.ModelSerializer):
    is_owner = serializers.SerializerMethodField()
    can_edit = serializers.SerializerMethodField()
    url = serializers.SerializerMethodField()
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'is_owner', 'can_edit', 'url']
    
    def get_is_owner(self, obj):
        """
        Execution: For each object during serialization
        Context available: request, view, format
        """
        request = self.context.get('request')
        if not request or not request.user.is_authenticated:
            return False
        return obj.author == request.user
    
    def get_can_edit(self, obj):
        request = self.context.get('request')
        view = self.context.get('view')
        
        # Check permission without making extra queries
        permission = IsAuthorOrReadOnly()
        return permission.has_object_permission(request, view, obj)
    
    def get_url(self, obj):
        request = self.context.get('request')
        # Build absolute URL
        return request.build_absolute_uri(f'/api/posts/{obj.id}/')


# In view, context is automatically passed
class PostViewSet(viewsets.ModelViewSet):
    def list(self, request):
        posts = self.get_queryset()
        serializer = self.get_serializer(posts, many=True)
        # Context automatically includes: {'request': request, 'view': self, 'format': None}
        return Response(serializer.data)
```

**Context Flow:**
```
ViewSet.get_serializer()
  ↓
  context = {
      'request': self.request,
      'format': self.format_kwarg,
      'view': self
  }
  ↓
  Serializer(__init__, context=context)
  ↓
  Available in SerializerMethodField via self.context
```

---

## Practical Efficiency Guidelines

### Rule 1: Start Simple, Optimize When Measured

```python
# ❌ PREMATURE OPTIMIZATION (Don't do this initially)
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        # Too complex from the start
        return Post.objects.select_related(
            'author', 'author__profile', 'category'
        ).prefetch_related(
            Prefetch('comments', queryset=Comment.objects.select_related('user')),
            Prefetch('tags', queryset=Tag.objects.only('name')),
            'likes', 'likes__user'
        ).annotate(
            comments_count=Count('comments'),
            likes_count=Count('likes')
        ).only('id', 'title', 'author__username')

# ✅ START SIMPLE
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

# ✅ MEASURE PERFORMANCE
# Use Django Debug Toolbar or django-silk
# Identify N+1 queries

# ✅ OPTIMIZE SPECIFIC PROBLEMS
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        queryset = Post.objects.all()
        
        # Optimize only what you measured
        if self.action == 'list':
            queryset = queryset.select_related('author')
        
        return queryset
```

### Rule 2: Choose the Right Tool

```python
# Use Case: Simple CRUD with standard behavior
# ✅ BEST: ModelViewSet (minimal code)
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer


# Use Case: Need only list and detail, no create/update/delete
# ✅ BEST: ReadOnlyModelViewSet
class PostViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer


# Use Case: Custom business logic, not standard CRUD
# ✅ BEST: APIView
class ExportPostsView(APIView):
    def post(self, request):
        # Custom logic that doesn't fit CRUD pattern
        format = request.data.get('format')
        posts = Post.objects.all()
        
        if format == 'csv':
            return self.export_csv(posts)
        elif format == 'pdf':
            return self.export_pdf(posts)
        
        return Response({'error': 'Invalid format'}, status=400)


# Use Case: Single utility endpoint
# ✅ BEST: Function-based view
@api_view(['POST'])
@permission_classes([IsAdminUser])
def rebuild_search_index(request):
    from tasks import rebuild_index
    rebuild_index.delay()
    return Response({'status': 'started'})


# Use Case: Complex multi-step operation
# ✅ BEST: Custom ViewSet method
class PostViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    def publish_and_notify(self, request, pk=None):
        """
        Complex operation: publish post and send notifications
        Not a standard CRUD operation
        """
        post = self.get_object()
        post.status = 'published'
        post.published_at = timezone.now()
        post.save()
        
        # Send notifications
        notify_followers.delay(post.id)
        
        return Response({
            'status': 'published',
            'published_at': post.published_at
        })
```

### Rule 3: Understand Query Execution

```python
# Database queries execute at EVALUATION time, not creation time

# ❌ INEFFICIENT: Queries execute in loop
def get_posts_with_author_names():
    posts = Post.objects.all()  # Query NOT executed yet (lazy)
    result = []
    
    for post in posts:  # Query executes here
        author_name = post.author.username  # N+1: New query for each post!
        result.append({
            'title': post.title,
            'author': author_name
        })
    
    return result

# Execution:
# Query 1: SELECT * FROM posts
# Query 2: SELECT * FROM users WHERE id = 1
# Query 3: SELECT * FROM users WHERE id = 2
# ... 100 queries for 100 posts!


# ✅ EFFICIENT: Single query with join
def get_posts_with_author_names():
    posts = Post.objects.select_related('author').all()
    result = []
    
    for post in posts:  # Still only ONE query
        author_name = post.author.username  # No additional query
        result.append({
            'title': post.title,
            'author': author_name
        })
    
    return result

# Execution:
# Query 1: SELECT * FROM posts 
#          INNER JOIN users ON posts.author_id = users.id
# Total: 1 query


# Understanding query execution timing:
queryset = Post.objects.all()  # NO query yet
queryset = queryset.filter(published=True)  # Still NO query
queryset = queryset.select_related('author')  # STILL no query

# Query executes when you:
list(queryset)  # Convert to list
queryset[0]  # Access by index
for post in queryset:  # Iterate
    pass
len(queryset)  # Get count
bool(queryset)  # Check if exists
```

### Rule 4: DRY (Don't Repeat Yourself) vs Explicit

```python
# Balance between DRY and explicit code

# ❌ TOO DRY (Hard to understand)
class BaseSerializer(serializers.ModelSerializer):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # Magic happens here, but unclear what
        self.process_dynamic_fields()
        self.apply_conditional_validation()
        self.setup_context_based_fields()
    
    # ... 200 lines of abstraction

class PostSerializer(BaseSerializer):
    # No idea what this does without reading BaseSerializer
    class Meta:
        model = Post
        fields = '__all__'


# ❌ NOT DRY ENOUGH (Repetitive)
class PostCreateSerializer(serializers.ModelSerializer):
    def validate_title(self, value):
        if len(value) < 5:
            raise serializers.ValidationError("Title too short")
        if 'spam' in value.lower():
            raise serializers.ValidationError("Spam detected")
        return value
    
    class Meta:
        model = Post
        fields = ['title', 'content']

class PostUpdateSerializer(serializers.ModelSerializer):
    def validate_title(self, value):
        # Same validation repeated!
        if len(value) < 5:
            raise serializers.ValidationError("Title too short")
        if 'spam' in value.lower():
            raise serializers.ValidationError("Spam detected")
        return value
    
    class Meta:
        model = Post
        fields = ['title', 'content', 'status']


# ✅ BALANCED: DRY but still explicit
class TitleValidationMixin:
    """Clear, reusable validation"""
    def validate_title(self, value):
        if len(value) < 5:
            raise serializers.ValidationError("Title too short")
        if 'spam' in value.lower():
            raise serializers.ValidationError("Spam detected")
        return value

class PostCreateSerializer(TitleValidationMixin, serializers.ModelSerializer):
    # Clear what it does, validation is separate
    class Meta:
        model = Post
        fields = ['title', 'content']

class PostUpdateSerializer(TitleValidationMixin, serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['title', 'content', 'status']
```

---

## Common Mistakes & Solutions

### Mistake 1: Over-fetching Data

```python
# ❌ WRONG: Fetching unnecessary data
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = '__all__'  # Returns ALL fields including large content

# API returns:
{
    "id": 1,
    "title": "Post",
    "content": "... 10,000 characters ...",  # Wasted bandwidth for list view
    "raw_html": "...",  # Not needed
    "internal_notes": "...",  # Should never be exposed
    "password_hash": "..."  # SECURITY ISSUE!
}


# ✅ CORRECT: Explicit fields for different contexts
class PostListSerializer(serializers.ModelSerializer):
    """Minimal data for list view"""
    class Meta:
        model = Post
        fields = ['id', 'title', 'excerpt', 'created_at']  # Only what's needed

class PostDetailSerializer(serializers.ModelSerializer):
    """Full data for detail view"""
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'author', 'created_at', 'updated_at']
        # Explicit exclusion is safer than fields = '__all__'


# ✅ OPTIMAL: Use .only() for database efficiency
class PostViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        queryset = Post.objects.all()
        
        if self.action == 'list':
            # Only fetch fields we'll serialize
            queryset = queryset.only('id', 'title', 'excerpt', 'created_at')
        
        return queryset
```

### Mistake 2: Not Handling Errors Properly

```python
# ❌ WRONG: Generic error handling
class PostViewSet(viewsets.ModelViewSet):
    def create(self, request):
        try:
            serializer = PostSerializer(data=request.data)
            serializer.save()  # No validation!
            return Response(serializer.data)
        except Exception as e:
            return Response({'error': 'Something went wrong'}, status=500)
        # Lost all error details!


# ✅ CORRECT: Let DRF handle validation errors
class PostViewSet(viewsets.ModelViewSet):
    def create(self, request):
        serializer = PostSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)  # Raises ValidationError with details
        serializer.save()
        return Response(serializer.data, status=status.HTTP_201_CREATED)


# ✅ BETTER: Handle specific exceptions explicitly
class PostViewSet(viewsets.ModelViewSet):
    def create(self, request):
        serializer = PostSerializer(data=request.data)
        
        try:
            serializer.is_valid(raise_exception=True)
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        
        except serializers.ValidationError:
            # Let DRF handle validation errors (returns 400 with details)
            raise
        
        except IntegrityError as e:
            # Database constraint violation
            return Response(
                {'error': 'Duplicate entry or constraint violation'},
                status=status.HTTP_409_CONFLICT
            )
        
        except PermissionDenied:
            # Re-raise permission errors (returns 403)
            raise
        
        except Exception as e:
            # Log unexpected errors
            logger.exception("Unexpected error in post creation")
            return Response(
                {'error': 'An unexpected error occurred'},
                status=status.HTTP_500_INTERNAL_SERVER_ERROR
            )
```

### Mistake 3: Not Using Transactions

```python
# ❌ WRONG: Multiple save operations without transaction
class PostViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    def publish_with_notification(self, request, pk=None):
        post = self.get_object()
        
        # If this succeeds...
        post.status = 'published'
        post.save()
        
        # But this fails...
        notification = Notification.objects.create(
            user=post.author,
            message=f"Your post '{post.title}' is published"
        )  # CRASH! But post is already saved!
        
        # Database is now inconsistent: post published but no notification
        return Response({'status': 'published'})


# ✅ CORRECT: Use atomic transactions
from django.db import transaction

class PostViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    @transaction.atomic  # Decorator approach
    def publish_with_notification(self, request, pk=None):
        post = self.get_object()
        
        post.status = 'published'
        post.save()
        
        Notification.objects.create(
            user=post.author,
            message=f"Your post '{post.title}' is published"
        )
        
        # If ANY exception occurs, BOTH operations rollback
        return Response({'status': 'published'})
    
    # Alternative: Context manager approach
    def create(self, request):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        
        with transaction.atomic():
            # All database operations in this block are atomic
            post = serializer.save()
            
            # Additional operations
            post.author.post_count += 1
            post.author.save()
            
            Tag.objects.bulk_create([
                Tag(post=post, name=tag)
                for tag in request.data.get('tags', [])
            ])
        
        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

### Mistake 4: Ignoring Caching Opportunities

```python
# ❌ INEFFICIENT: Hitting database on every request
class PostViewSet(viewsets.ModelViewSet):
    def retrieve(self, request, pk=None):
        post = Post.objects.get(pk=pk)
        serializer = self.get_serializer(post)
        return Response(serializer.data)
    # Every request hits database, even for unchanged data


# ✅ GOOD: Basic caching
from django.core.cache import cache

class PostViewSet(viewsets.ModelViewSet):
    def retrieve(self, request, pk=None):
        cache_key = f'post:{pk}'
        cached_data = cache.get(cache_key)
        
        if cached_data:
            return Response(cached_data)
        
        post = Post.objects.get(pk=pk)
        serializer = self.get_serializer(post)
        
        # Cache for 15 minutes
        cache.set(cache_key, serializer.data, 60 * 15)
        
        return Response(serializer.data)
    
    def update(self, request, pk=None):
        # Invalidate cache on update
        cache.delete(f'post:{pk}')
        return super().update(request, pk=pk)


# ✅ BETTER: Versioned caching (cache invalidation)
class PostViewSet(viewsets.ModelViewSet):
    def retrieve(self, request, pk=None):
        post = Post.objects.only('updated_at').get(pk=pk)
        
        # Cache key includes version (updated_at timestamp)
        cache_key = f'post:{pk}:{post.updated_at.timestamp()}'
        cached_data = cache.get(cache_key)
        
        if cached_data:
            return Response(cached_data)
        
        # Re-fetch with all fields
        post = Post.objects.get(pk=pk)
        serializer = self.get_serializer(post)
        
        # Cache indefinitely (new version will have different key)
        cache.set(cache_key, serializer.data, None)
        
        return Response(serializer.data)


# ✅ BEST: Use Django's cache decorators
from django.views.decorators.cache import cache_page
from django.utils.decorators import method_decorator

class PostViewSet(viewsets.ModelViewSet):
    @method_decorator(cache_page(60 * 15))  # Cache for 15 minutes
    def list(self, request):
        return super().list(request)
```

### Mistake 5: Inefficient Pagination

```python
# ❌ WRONG: No pagination (returns ALL records)
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()  # Could be 100,000 records!
    serializer_class = PostSerializer
    # Client receives HUGE response, slow serialization


# ⚠️ PROBLEMATIC: PageNumberPagination for large datasets
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    pagination_class = PageNumberPagination
    
    # Problem: Offset pagination is slow for deep pages
    # SELECT * FROM posts LIMIT 20 OFFSET 10000
    # Database must scan 10,020 rows to return 20


# ✅ GOOD: CursorPagination for large datasets
from rest_framework.pagination import CursorPagination

class PostCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    pagination_class = PostCursorPagination
    
    # Uses WHERE created_at < '...' instead of OFFSET
    # Much faster for large datasets


# ✅ OPTIMAL: Custom pagination with optimization
class OptimizedPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100
    
    def get_paginated_response(self, data):
        return Response({
            'count': self.page.paginator.count,
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'results': data,
            # Additional metadata
            'page_size': self.page_size,
            'current_page': self.page.number,
            'total_pages': self.page.paginator.num_pages
        })
```

---

## Testing Strategies

### Unit Testing Serializers

```python
from django.test import TestCase
from rest_framework.test import APITestCase

class PostSerializerTest(TestCase):
    def setUp(self):
        self.user = User.objects.create(username='testuser')
        self.post = Post.objects.create(
            title='Test Post',
            content='Test content',
            author=self.user
        )
    
    def test_serialization(self):
        """Test model → dict conversion"""
        serializer = PostSerializer(self.post)
        data = serializer.data
        
        self.assertEqual(data['title'], 'Test Post')
        self.assertEqual(data['content'], 'Test content')
        self.assertEqual(data['author_id'], self.user.id)
    
    def test_deserialization_valid(self):
        """Test dict → model with valid data"""
        data = {
            'title': 'New Post',
            'content': 'New content',
            'author_id': self.user.id
        }
        
        serializer = PostSerializer(data=data)
        self.assertTrue(serializer.is_valid())
        post = serializer.save()
        
        self.assertEqual(post.title, 'New Post')
        self.assertIsInstance(post, Post)
    
    def test_validation_title_too_short(self):
        """Test validation error"""
        data = {
            'title': 'Hi',  # Too short
            'content': 'Content',
            'author_id': self.user.id
        }
        
        serializer = PostSerializer(data=data)
        self.assertFalse(serializer.is_valid())
        self.assertIn('title', serializer.errors)
    
    def test_read_only_fields(self):
        """Test that read-only fields can't be written"""
        data = {
            'id': 999,  # Should be ignored
            'title': 'Post',
            'content': 'Content',
            'author_id': self.user.id
        }
        
        serializer = PostSerializer(data=data)
        serializer.is_valid(raise_exception=True)
        post = serializer.save()
        
        self.assertNotEqual(post.id, 999)  # id was auto-generated
```

### Integration Testing ViewSets

```python
from rest_framework.test import APITestCase, APIClient
from rest_framework import status

class PostViewSetTest(APITestCase):
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        # Authenticate for protected endpoints
        self.client.force_authenticate(user=self.user)
        
        self.post = Post.objects.create(
            title='Test Post',
            content='Test content',
            author=self.user
        )
    
    def test_list_posts(self):
        """Test GET /api/posts/"""
        response = self.client.get('/api/posts/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 1)
        self.assertEqual(response.data['results'][0]['title'], 'Test Post')
    
    def test_create_post(self):
        """Test POST /api/posts/"""
        data = {
            'title': 'New Post',
            'content': 'New content'
        }
        
        response = self.client.post('/api/posts/', data)
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Post.objects.count(), 2)
        self.assertEqual(response.data['title'], 'New Post')
    
    def test_create_post_unauthenticated(self):
        """Test POST /api/posts/ without authentication"""
        self.client.force_authenticate(user=None)  # Remove authentication
        
        data = {'title': 'New Post', 'content': 'Content'}
        response = self.client.post('/api/posts/', data)
        
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
    
    def test_update_own_post(self):
        """Test PUT /api/posts/{id}/ - owner can update"""
        data = {'title': 'Updated Title', 'content': 'Updated content'}
        
        response = self.client.put(f'/api/posts/{self.post.id}/', data)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.post.refresh_from_db()
        self.assertEqual(self.post.title, 'Updated Title')
    
    def test_update_others_post(self):
        """Test PUT /api/posts/{id}/ - non-owner cannot update"""
        other_user = User.objects.create_user(
            username='otheruser',
            password='pass123'
        )
        self.client.force_authenticate(user=other_user)
        
        data = {'title': 'Hacked Title', 'content': 'Hacked'}
        response = self.client.put(f'/api/posts/{self.post.id}/', data)
        
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
    
    def test_delete_post(self):
        """Test DELETE /api/posts/{id}/"""
        response = self.client.delete(f'/api/posts/{self.post.id}/')
        
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertEqual(Post.objects.count(), 0)
    
    def test_custom_action(self):
        """Test POST /api/posts/{id}/publish/"""
        response = self.client.post(f'/api/posts/{self.post.id}/publish/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.post.refresh_from_db()
        self.assertTrue(self.post.published)


class PostQueryOptimizationTest(APITestCase):
    """Test query count to ensure optimization"""
    
    def setUp(self):
        self.user = User.objects.create_user(username='user', password='pass')
        self.client.force_authenticate(user=self.user)
        
        # Create test data
        for i in range(10):
            Post.objects.create(
                title=f'Post {i}',
                content=f'Content {i}',
                author=self.user
            )
    
    def test_list_query_count(self):
        """Ensure list view doesn't cause N+1 queries"""
        with self.assertNumQueries(2):  # 1 for posts, 1 for count
            response = self.client.get('/api/posts/')
            self.assertEqual(len(response.data['results']), 10)
```

---

## Performance Optimization Checklist

### Database Level

```python
# ✅ 1. Use select_related for ForeignKey
Post.objects.select_related('author', 'category')

# ✅ 2. Use prefetch_related for ManyToMany and reverse ForeignKey
Post.objects.prefetch_related('tags', 'comments')

# ✅ 3. Use only() to fetch specific fields
Post.objects.only('id', 'title', 'created_at')

# ✅ 4. Use defer() to exclude heavy fields
Post.objects.defer('content', 'raw_html')

# ✅ 5. Use values() for dictionaries (faster than model instances)
Post.objects.values('id', 'title')  # Returns dict, not model

# ✅ 6. Use database indexes
class Post(models.Model):
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['author', 'created_at']),
            models.Index(fields=['-created_at']),  # For ordering
        ]

# ✅ 7. Use bulk operations
Post.objects.bulk_create([
    Post(title=f'Post {i}', content=f'Content {i}')
    for i in range(1000)
])  # Single query instead of 1000

# ✅ 8. Use update() instead of save() for bulk updates
Post.objects.filter(author=user).update(status='published')
# Instead of:
# for post in Post.objects.filter(author=user):
#     post.status = 'published'
#     post.save()
```

### Serializer Level

```python
# ✅ 1. Use SerializerMethodField sparingly (expensive)
class PostSerializer(serializers.ModelSerializer):
    # ❌ BAD: Causes N+1
    comments_count = serializers.SerializerMethodField()
    
    def get_comments_count(self, obj):
        return obj.comments.count()  # Query for EACH post
    
    # ✅ GOOD: Annotate in queryset
    # In ViewSet:
    # queryset = Post.objects.annotate(comments_count=Count('comments'))
    comments_count = serializers.IntegerField(read_only=True)

# ✅ 2. Use source for simple field access
class PostSerializer(serializers.ModelSerializer):
    author_name = serializers.CharField(source='author.username', read_only=True)
    # Faster than SerializerMethodField

# ✅ 3. Avoid nested writes unless necessary
# Nested reads are fine, but nested writes add complexity

# ✅ 4. Use ListSerializer for bulk operations
class PostListSerializer(serializers.ListSerializer):
    def create(self, validated_data):
        posts = [Post(**item) for item in validated_data]
        return Post.objects.bulk_create(posts)

class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = '__all__'
        list_serializer_class = PostListSerializer
```

### View Level

```python
# ✅ 1. Implement caching
@method_decorator(cache_page(60 * 15))
def list(self, request):
    return super().list(request)

# ✅ 2. Use pagination
pagination_class = CursorPagination

# ✅ 3. Implement filtering to reduce dataset
filter_backends = [DjangoFilterBackend, SearchFilter]
filterset_fields = ['status', 'author']

# ✅ 4. Use throttling to prevent abuse
throttle_classes = [UserRateThrottle]

# ✅ 5. Optimize queryset per action
def get_queryset(self):
    queryset = super().get_queryset()
    
    if self.action == 'list':
        queryset = queryset.select_related('author').only('id', 'title')
    elif self.action == 'retrieve':
        queryset = queryset.select_related('author').prefetch_related('comments')
    
    return queryset
```

---

## Summary: Decision Tree

```
Need an API endpoint?
│
├─ Standard CRUD operations?
│  │
│  ├─ YES → Use ModelViewSet
│  │      queryset = Model.objects.all()
│  │      serializer_class = ModelSerializer
│  │
│  └─ NO → Custom logic?
│     │
│     ├─ Multiple operations → Use APIView
│     └─ Single operation → Use @api_view
│
├─ Need authentication?
│  │
│  ├─ Token-based → authentication_classes = [TokenAuthentication]
│  ├─ Session-based → authentication_classes = [SessionAuthentication]
│  └─ JWT → Use djangorestframework-simplejwt
│
├─ Need permissions?
│  │
│  ├─ Simple → permission_classes = [IsAuthenticated]
│  └─ Complex → Create custom permission class
│
├─ Performance concerns?
│  │
│  ├─ Database → Use select_related, prefetch_related, only()
│  ├─ Serialization → Different serializers per action
│  └─ Response time → Implement caching
│
└─ Testing?
   │
   ├─ Serializers → TestCase with serializer.is_valid()
   └─ Views → APITestCase with self.client.get/post()
```

---

## Final Principles

### 1. **First Principles: Build from Foundation**
- HTTP can only send text
- Need conversion: Objects ↔ Text
- Need validation: Ensure data integrity
- Need abstraction: Reduce boilerplate
- **Result**: Serializers + Views + Routers

### 2. **Circle of Competence: Master Gradually**
- **Level 1**: ModelViewSet + ModelSerializer (80% of use cases)
- **Level 2**: Custom actions, permissions, filtering
- **Level 3**: Performance optimization, custom renderers

### 3. **Efficiency: Measure, Then Optimize**
- Start simple
- Measure performance (Django Debug Toolbar)
- Optimize bottlenecks
- Don't premature optimize

### 4. **Code Quality: Balance DRY and Clarity**
- Reuse common patterns (mixins, base classes)
- Keep code explicit and readable
- Document non-obvious decisions
- Write tests for critical paths

---

## Quick Reference

```python
# Minimal working API
from rest_framework import viewsets, serializers
from rest_framework.routers import DefaultRouter

class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'title', 'content']

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

router = DefaultRouter()
router.register(r'posts', PostViewSet)
urlpatterns = router.urls
```

**That's it!** This gives you a fully functional REST API with:
- GET /posts/ (list)
- POST /posts/ (create)
- GET /posts/{id}/ (retrieve)
- PUT /posts/{id}/ (update)
- PATCH /posts/{id}/ (partial update)
- DELETE /posts/{id}/ (delete)

Build from here based on your specific needs.

---

## Additional Resources

- **Official Docs**: https://www.django-rest-framework.org/
- **Source Code**: https://github.com/encode/django-rest-framework
- **Django ORM**: https://docs.djangoproject.com/en/stable/topics/db/queries/
- **Testing**: https://www.django-rest-framework.org/api-guide/testing/

Remember: **Start simple, measure performance, optimize when needed**.
